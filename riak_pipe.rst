=========
Riak Pipe
=========

Riak Source Code Reading @Tokyo #11

:author: Chiharu Kawatake (github: mille-printemps)
:date: 2013-05-07
:riak_pipe: 7d8e638cf1e5da44e440df6c7c58d0ea95936d52 Merge branch '1.3'

.. contents:: :depth: 3

%% definitions of pipeline, sink, fitting, etc...


サーバ
======

riak_pipe_app.erl
-----------------

* application として実装されている

``riak_pipe_app:start/2``::

    start(_StartType, _StartArgs) ->
        %% startup mostly copied from riak_kv
        catch cluster_info:register_app(riak_pipe_cinfo),

        case riak_pipe_sup:start_link() of             % riak_pipe の supervisor を開始
            {ok, Pid} ->
                riak_core:register(riak_pipe, [        % vnode module として riak_pipe_vnode を設定
                 {vnode_module, riak_pipe_vnode},      % これにより vnode へのコマンドは 
                 {stat_mod, riak_pipe_stat}            % riak_core_vnode を経由して riak_pipe_vnode で処理される
                ]),
                {ok, Pid};
            {error, Reason} ->
                {error, Reason}
        end.

        
riak_pipe_sup.erl
-----------------

* supervisor として実装されている

``riak_pipe_sup:init/1``::

    init([]) ->
        %% ordsets = enabled traces are represented as ordsets in fitting_details
        %% sets = '' sets ''
        riak_core_capability:register(
          {riak_pipe, trace_format}, [ordsets, sets], sets),

        VMaster = {riak_pipe_vnode_master,                                 % riak_core_vnode_master を開始
                   {riak_core_vnode_master, start_link, [riak_pipe_vnode]},
                   permanent, 5000, worker, [riak_core_vnode_master]},
                   
        BSup = {riak_pipe_builder_sup,                                     % riak_pipe_builder の supervisor を開始
                {riak_pipe_builder_sup, start_link, []},                   % restart strategy が simple_one_for_one
                   permanent, 5000, supervisor, [riak_pipe_builder_sup]},  % riak_pipe_builder は start_child/2 で開始
                                                                           % 以下の supervisor も同様
                   
        FSup = {riak_pipe_fitting_sup,                                     % riak_pipe_fitting の supervisor を開始
                {riak_pipe_fitting_sup, start_link, []},                     
                permanent, 5000, supervisor, [riak_pipe_fitting_sup]},       
                
        CSup = {riak_pipe_qcover_sup,                                      % riak_qcover の supervisor を開始
                {riak_pipe_qcover_sup, start_link, []},                      
                permanent, 5000, supervisor, [riak_pipe_qcover_sup]},        
                
        {ok, { {one_for_one, 5, 10}, [VMaster, BSup, FSup, CSup]} }.       % one_for_one
                                                                           % 子プロセスが落ちた場合そのプロセスのみ再開

クライアント
============

riak_pipe.erl
-------------

* クライアントの API を定義
* クライアントが主に使用する API は以下のもの

    - riak_pipe:exec/2 -> pipeline の構築
    - riak_pipe:queue_work -> 入力の送信
    - riak_pipe:collect_results/1, riak_pipe:collect_result/1 -> 結果の受信
    
* 簡単なサンプルの実装あり。


pipeline の構築
---------------

riak_pipe:exec/2
~~~~~~~~~~~~~~~~
* riak_pipe_builder を使って pipeline を構築する

    - riak_pipe_builder を開始
    - riak_pipe_fitting を開始
    - riak_pipe_builder と riak_pipe_fitting は子プロセスとして動的に追加される
    
        + supervisor が落ちて再開されても子プロセスは自動的に再開されない
        + riak_pipe_builder と riak_pipe_fitting がお互いに erlang:monitor する実装になっている
        
* #pipe{} を返す

* サンプル - riak_pipe:example_start/0 より

::

    {ok, Pipe} = riak_pipe:exec(
                [#fitting_spec{name=empty_pass,
                     module=riak_pipe_w_pass,
                     chashfun=fun(_) -> <<0:160/integer>> end}],
                [{log, sink},
                 {trace, all}]).


``riak_pipe:exec/2``::

    exec(Spec, Options) ->
        [ riak_pipe_fitting:validate_fitting(F) || F <- Spec ],
        CorrectOptions = correct_trace(
                           validate_sink_type(
                             ensure_sink(Options))),             % Options が [] であった場合は生成される
                                                                 % [{sink, #fitting{pid=self(), ref=make_ref(), chashfun=sink}}]
                                                                 % となるので、Sink はクライアントプロセスになる
                                                              
    riak_pipe_builder_sup:new_pipeline(Spec, CorrectOptions).    % pipeline を構築


riak_pipe_builder_sup:new_pipeline/2
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
* riak_pipe_builder を開始
* riak_pipe_builder に pipeline イベントを送信
* #pipe{} を返す

::

    -record(pipe,
        {
          builder :: pid(),
          fittings :: [{Name::term(), #fitting{}}],
          sink :: #fitting{}
        }).

    -record(fitting,
        {
          pid :: pid(),                            % fitting の pid
          ref :: reference(),                      % fitting の reference
          chashfun :: riak_pipe_vnode:chashfun(),  % 入力をどのように vnode へ分散させるかを決める hash 関数
          nval :: riak_pipe_vnode:nval()           % 入力を処理する vnode の最大数
        }).
 

``riak_pipe_builder_sup:new_pipeline/2``::

    new_pipeline(Spec, Options) ->
        case supervisor:start_child(?MODULE, [Spec, Options]) of % riak_pipe_builder を開始して
                                                                 % supervisor の子プロセスとして追加
            {ok, Pid, Ref} ->
                case riak_pipe_builder:pipeline(Pid) of          % pipeline イベントを送信       
                    {ok, #pipe{sink=#fitting{ref=Ref}}=Pipe} ->
                        riak_pipe_stat:update({create, Pid}),    % 統計情報を収集
                        {ok, Pipe};                              % #pipe{} を返す -> exec の返り値
                    _ ->
                        riak_pipe_stat:update(create_error),
                        {error, startup_failure}
                end;
            Error ->
                riak_pipe_stat:update(create_error),
                Error
        end.

        
riak_pipe_builder:init/1
~~~~~~~~~~~~~~~~~~~~~~~~
* riak_pipe_builder は gen_fsm として実装されている
* Sink を開始する
* Fitting を開始する
* #pipe{} を生成

``riak_pipe_builder:init/1``::

    init([Spec, Options]) ->
        {sink, #fitting{ref=Ref}=Sink} = lists:keyfind(sink, 1, Options),
        
        SinkMon = erlang:monitor(process, Sink#fitting.pid),               % Sink を監視
        
        Fittings = start_fittings(Spec, Options),                          % Spec に指定された Fitting を開始
        NamedFittings = lists:zip(
                          [ N || #fitting_spec{name=N} <- Spec ],
                          [ F || {F, _R} <- Fittings ]),                   % [{<spec name>, #fitting{pid, ref, chashfun, nval}}, ...] を返す
                          
        Pipe = #pipe{builder=self(),
                     fittings=NamedFittings,
                     sink=Sink},                                           % exec の返り値となる #pipe{} を生成
                     
        put(eunit, [{module, ?MODULE},
                    {ref, Ref},
                    {spec, Spec},
                    {options, Options},
                    {fittings, Fittings}]),                                % pipe の情報を process dictionary へ格納
                                                                           % unit test に使う?
        {ok, wait_pipeline_shutdown,
        #state{options=Options,
                pipe=Pipe,
                alive=Fittings,
                sinkmon=SinkMon}}.                                         % wait_pipeline_shutdown へ遷移

                
``riak_pipe_builder:start_fittings/2``::

    start_fittings(Spec, Options) ->
        [Tail|Rest] = lists:reverse(Spec),                                 % Spec のリストを反転
        ClientOutput = client_output(Options),
        lists:foldl(fun(FitSpec, [{Output,_}|_]=Acc) ->
                            [start_fitting(FitSpec, Output, Options)|Acc]
                    end,
                    [start_fitting(Tail, ClientOutput, Options)],
                    Rest).                                                 % 反転した Spec に順に start_fitting/3 を適用
                                                                           % [#fitting{pid, ref, chashfun, nval}, Ref}, ...] 

``riak_pipe_builder:start_fitting/3``::
 
    start_fitting(Spec, Output, Options) ->
        ?DPF("Starting fitting for ~p", [Spec]),
        {ok, Pid, Fitting} = riak_pipe_fitting_sup:add_fitting(
                               self(), Spec, Output, Options),             % riak_pipe_fitting を開始
        Ref = erlang:monitor(process, Pid),                                % riak_pipe_fitting を監視
        
        {Fitting, Ref}.                                                    % {#fitting{pid, ref, chashfun, nval}, Ref}
                                                                           % pid は fitting の pid
                                                                           % ref は自分の次の fitting の ref
        
riak_pipe_fitting_sup:add_fitting/4
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
* riak_pipe_fitting を開始する
        
``riak_pipe_fitting_sup:add_fitting/4``::

    add_fitting(Builder, Spec, Output, Options) ->
        ?DPF("Adding fitting for ~p", [Spec]),
        supervisor:start_child(?SERVER, [Builder, Spec, Output, Options]). % riak_pipe_fitting を開始
                                                                           % supervisor の子プロセスとして追加

riak_pipe_fitting:init/1
~~~~~~~~~~~~~~~~~~~~~~~~
* riak_pipe_fitting は gen_fsm として実装されている
* riak_pipe:exec/2 で渡された #fitting_spec{} を保持する
* 状態を wait_upstream_eoi に遷移させる

``riak_pipe_fitting:init/1``::

    init([Builder,
          #fitting_spec{name=Name, module=Module, arg=Arg, q_limit=QLimit}=Spec,
          Output,
          Options]) ->
        Fitting = fitting_record(self(), Spec, Output),
        Details = #fitting_details{fitting=Fitting,
                                   name=Name,
                                   module=Module,
                                   arg=Arg,
                                   output=Output,
                                   options=Options,
                                   q_limit=QLimit},                  % #fitting_spec{} を保持

        ?T(Details, [], {fitting, init_started}),                    % riak_pipe_log.hrl に定義されているマクロ
                                                                     % riak_pipe_log:trace/3 を呼び出している
                                                                     
        erlang:monitor(process, Builder),                            % riak_pipe_builder を監視

        ?T(Details, [], {fitting, init_finished}),

        put(eunit, [{module, ?MODULE},
                    {fitting, Fitting},
                    {details, Details},
                    {builder, Builder}]),
                    
        {ok, wait_upstream_eoi,
         #state{builder=Builder, details=Details, workers=[],
            ref=Output#fitting.ref}}.                                % wait_upstream_eoi へ遷移

%% summary
            
入力の送信
----------

* ``riak_pipe:queue_work/2`` により fitting へ入力を送信。
* ``riak_pipe:queue:work/3`` から最終的に ``riak_pipe_vnode:queue:work/4`` が呼ばれる。
* ``riak_pipe_vnoce:queue:work/4`` は fitting spec に設定される chashfun (consistent-hashing function) により4通り定義されている。
* サンプル - riak_pipe:example_send/1 より

::

    ok = riak_pipe:queue_work(Pipe, "hello"),                        % riak_pipe:exec/2 から得た Pipe を渡して
                                                                     % "hello" を送信
    riak_pipe:eoi(Pipe).                                             % 入力の終了を fitting へ送信

    
riak_pipe:queue_work/3
~~~~~~~~~~~~~~~~~~~~~~
``riak_pipe:queue_work/3``::

    queue_work(#pipe{fittings=[{_,Head}|_]}, Input, Timeout)
      when Timeout =:= infinity; Timeout =:= noblock ->
        riak_pipe_vnode:queue_work(Head, Input, Timeout).            % 先頭の fitting (#fitting{}) を渡して
                                                                     % riak_pipe_vnode:queue_work/3 を呼ぶ
        
riak_pipe_vnode:queue_work/4
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
* Spec に設定された hash 関数に基づいて vnode 
* hash 関数による入力の分散の例は参考資料を参照

``riak_pipe_vnode:queue_work/4``::

    queue_work(#fitting{chashfun=follow}=Fitting,                    % デフォルトの場合(Option が [])もしくは
               Input, Timeout, UsedPreflist) ->                      % hash 関数に follow が設定されていた場合
        queue_work(Fitting, Input, Timeout, UsedPreflist, any_local_vnode());
        
    queue_work(#fitting{chashfun={Module, Function}}=Fitting,        % hash 関数が設定されていた場合
           Input, Timeout, UsedPreflist) ->
        queue_work(Fitting, Input, Timeout, UsedPreflist,
                   Module:Function(Input));
               
    queue_work(#fitting{chashfun=Hash}=Fitting,                      % hash 関数に固定値が設定されていた場合
           Input, Timeout, UsedPreflist) when not is_function(Hash) ->
            queue_work(Fitting, Input, Timeout, UsedPreflist, Hash);
            
    queue_work(#fitting{chashfun=HashFun}=Fitting,                   % 1.0.x との互換性のため
           Input, Timeout, UsedPreflist) ->
        %% 1.0.x compatibility
        Hash = riak_pipe_fun:compat_apply(HashFun, [Input]),
        queue_work(Fitting, Input, Timeout, UsedPreflist, Hash).

        
``riak_pipe_vnode:queue_work/5``::

    queue_work(Fitting, Input, Timeout, UsedPreflist, Hash) ->
        queue_work_erracc(Fitting, Input, Timeout, UsedPreflist, Hash, []). % queue_work_erracc へ委譲

        
``riak_pipe_vnode:queue_work_erracc/6``::

    queue_work_erracc(#fitting{nval=NVal}=Fitting,
        Input, Timeout, UsedPreflist, Hash, ErrAcc) ->
        
        case remaining_preflist(Input, Hash, NVal, UsedPreflist) of         % riak_core へ委譲して 
            [NextPref|_] ->                                                 % vnode のリストを取得
                case queue_work_send(Fitting, Input, Timeout,
                                     [NextPref|UsedPreflist]) of            % 入力を vnode へ送信
                    ok -> ok;
                    {error, Error} ->
                        queue_work_erracc(Fitting, Input, Timeout,
                                          [NextPref|UsedPreflist], Hash,
                                          [Error|ErrAcc])
                end;
            [] ->
                if ErrAcc == [] ->
                        %% may happen if a fitting worker asks to forward
                        %% the input, but there is no more preflist to
                        %% forward to
                        {error, [preflist_exhausted]};
                   true ->
                        {error, ErrAcc}
                end
        end.

        
``riak_pipe_vnode:queue_work_send/4``::

* riak_core_vnode_master へ委譲して、fitting、入力、送信側の pid といった情報を vnode へ送る
        
        queue_work_send(#fitting{ref=Ref}=Fitting,
                    Input, Timeout,
                    [{Index,Node}|_]=UsedPreflist) ->
                    
            try riak_core_vnode_master:command_return_vnode(
                {Index, Node},
                #cmd_enqueue{fitting=Fitting, input=Input, timeout=Timeout,  
                            usedpreflist=UsedPreflist},
                {raw, Ref, self()},                                          
                riak_pipe_vnode_master) of                                   % vnode への情報の送信

                {ok, VnodePid} ->
                    queue_work_wait(Ref, Index, VnodePid);
                    
                {error, timeout} ->
                    {error, {vnode_proxy_timeout, {Index, Node}}}
                    
            catch exit:{{nodedown, Node}, _GenServerCall} ->
                    %% node died between services check and gen_server:call
                    {error, {nodedown, Node}}
            end.

            
riak_core_vnode_master:command_return_vnode/4
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
* riak_core_vnode_proxy へ委譲
* riak_core_vnode_proxy:handle_call/3 で実際に vnode へ情報が送信される

``riak_core_vnode_master:command_return_vnode/4``::

        command_return_vnode({Index,Node}, Msg, Sender, VMaster) ->
        
            % Req=#riak_vnode_req_v1{index, sender, request=#cmd_enqueue{}}        
            Req = make_request(Msg, Sender, Index),    
            
            case riak_core_capability:get({riak_core, vnode_routing}, legacy) of

                % legacy の場合も riak_core_vnode_proxy:command_return_vnode/2 を呼んでいる
                legacy ->
                    gen_server:call({VMaster, Node}, {return_vnode, Req});    
                    
                proxy ->
                     Mod = vmaster_to_vmod(VMaster),
                     riak_core_vnode_proxy:command_return_vnode({Mod,Index,Node}, Req)   
        end.

        
``riak_core_vnode_master:handle_call/3``::

    handle_call({return_vnode, Req}, _From, State) ->
        {Pid, NewState} = get_vnode_pid(State),
        
        gen_fsm:send_event(Pid, Req),        % Req で示されるイベントを Pid で示される vnode へ送信
        
        {reply, {ok, Pid}, NewState};

* vnode へ送信されるイベント

%%% explanation
        
::

    -type sender_type() :: fsm | server | raw.
    -type sender() :: {sender_type(), reference(), pid()} |
                       %% TODO: Double-check that these special cases are kosher
                      {server, undefined, undefined} | % special case in
                                                       % riak_core_vnode_master.erl
                      {fsm, undefined, pid()} |        % special case in
                                                       % riak_kv_util:make_request/2.erl
                      ignore.
    -type partition() :: non_neg_integer().
    -type vnode_req() :: term().

    -record(riak_vnode_req_v1, {
          index :: partition(),
          sender=ignore :: sender(),
          request :: vnode_req()}).

::
          
    vnode_req() = #cmd_enqueue{fitting=Fitting,
                              input=Input,
                              timeout=Timeout,
                              usedpreflist=UsedPreflist}

                              
riak_core_vnode:handle_event/3
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
* riak_core_vnode のイベント・ハンドラーが呼ばれる
* 上記の gen_fsm:send_event/2 により送信されたイベントに以下のハンドラーが適合する

``riak_core_vnode:handle_event/3``::

    ...
    handle_event(R=?VNODE_REQ{}, _StateName, State) ->
        active(R, State);
    ...
    
* handoff node が設定されているか否かにより、下記の riak_core_vnode:active/2 のどちらかが呼ばれる
* いずれにしても riak_pipe_vnode:handle_command/3 が呼ばれる
    
``riak_core_vnode:active/2``::
    
    ...
    active(?VNODE_REQ{sender=Sender, request=Request},
        State=#state{handoff_node=HN}) when HN =:= none ->
        vnode_command(Sender, Request, State);
        
    active(?VNODE_REQ{sender=Sender, request=Request},State) ->
        vnode_handoff_command(Sender, Request, State);
    ...

``riak_core_vnode:vnode_command/3``::

    vnode_command(Sender, Request, State=#state{index=Index,
                                            mod=Mod,
                                            modstate=ModState,
                                            forward=Forward,
                                            pool_pid=Pool}) ->
        %% Check if we should forward
        case Forward of
            undefined ->
                Action = Mod:handle_command(Request, Sender, ModState);    % Mod は riak_pipe_app:start/2 で設定された
                                                                           % riak_pipe_vnode
                
            NextOwner ->
                lager:debug("Forwarding ~p -> ~p: ~p~n", [node(), NextOwner, Index]),
                riak_core_vnode_master:command({Index, NextOwner}, Request, Sender,
                                           riak_core_vnode_master:reg_name(Mod)),
                Action = continue
        end,
        case Action of
            continue ->
                continue(State, ModState);
            {reply, Reply, NewModState} ->
                reply(Sender, Reply),                                      % ok を riak_pipe_vnode へ送信
                continue(State, NewModState);                              % active へ遷移
            {noreply, NewModState} ->
                continue(State, NewModState);
            {async, Work, From, NewModState} ->
                %% dispatch some work to the vnode worker pool
                %% the result is sent back to 'From'
                riak_core_vnode_worker_pool:handle_work(Pool, Work, From),
                continue(State, NewModState);
            {stop, Reason, NewModState} ->
                {stop, Reason, State#state{modstate=NewModState}}
        end.

riak_pipe_vnode:handle_command/3
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Request には #cmd_enqueue が指定されているので、下記の関数が適合する


``riak_pipe_vnode:handle_command/3``::        

    ...                                              
    handle_command(#cmd_enqueue{}=Cmd, Sender, State) ->
        enqueue_internal(Cmd, Sender, State);
    ...

* worker を生成して入力を worker のキューへ追加する
    
``riak_pipe_vnode:enqueue_internal/3``::

    enqueue_internal(#cmd_enqueue{fitting=Fitting, input=Input, timeout=TO,
                              usedpreflist=UsedPreflist},
                 Sender, #state{partition=Partition}=State) ->
                 
        case worker_for(Fitting, true, State) of                                % fitting に適合した worker を探す
                                                                                % 見つからない場合は新たに worker を生成
                                                                                
            {ok, #worker{details=#fitting_details{module=riak_pipe_w_crash}}}   % テスト用
              when Input == vnode_killer ->
              
                %% this is used by the eunit test named "Vnode Death"
                %% in riak_pipe:exception_test_; it kills the vnode before
                %% it has a chance to reply to the queue request
                exit({riak_pipe_w_crash, vnode_killer});
                
            {ok, Worker} when (Worker#worker.details)#fitting_details.module    % fitting のモジュールが 
                              /= ?FORWARD_WORKER_MODULE ->                      % riak_pipe_w_fwd でない場合
                              
                case add_input(Worker, Input, Sender, TO, UsedPreflist) of      % 入力を worker のキューへ追加
                
                    {ok, NewWorker} ->
                        ?T(NewWorker#worker.details, [queue],
                           {vnode, {queued, Partition, Input}}),
                        {reply, ok, replace_worker(NewWorker, State)};          % worker を riak_pipe_vnode の State へ追加
                        
                    {queue_full, NewWorker} ->
                        ?T(NewWorker#worker.details, [queue,queue_full],
                           {vnode, {queue_full, Partition, Input}}),
                        %% if the queue is full, hold up the producer
                        %% until we're ready for more
                        {noreply, replace_worker(NewWorker, State)};
                        
                    timeout ->
                        {reply, {error, timeout}, replace_worker(Worker, State)}
                end;
                
            {ok, _RestartForwardingWorker} ->
                %% this is a forwarding worker for a failed-restart
                %% fitting - don't enqueue any more inputs, just reject
                %% and let the requester enqueue elswhere
                {reply, {error, forwarding}, State};
                
            worker_limit_reached ->
                %% TODO: log/trace this event
                %% Except we don't have details here to associate with a trace
                %% function: ?T_ERR(WhereToGetDetails, whatever_limit_hit_here),
                {reply, {error, worker_limit_reached}, State};
                
            worker_startup_failed ->
                %% TODO: log/trace this event
                {reply, {error, worker_startup_failed}, State}
        end.

``riak_pipe_vnode:worker_for/3``::

    worker_for(Fitting, EnforceLimitP,
               #state{workers=Workers, worker_limit=Limit}=State) ->
        case worker_by_fitting(Fitting, State) of                           % State の worker から Fitting に適合するものを探す
            {ok, Worker} ->
                {ok, Worker};
            none ->
                if (not EnforceLimitP) orelse length(Workers) < Limit ->
                        new_worker(Fitting, State);                         % worker を新たに生成
                   true ->
                        worker_limit_reached
                end
        end.

``riak_pipe_vnode:new_worker/2``::

    new_worker(Fitting, #state{partition=P, worker_sup=Sup, worker_q_limit=WQL}) ->
        try
            case riak_pipe_fitting:get_details(Fitting, P) of               % fitting の pid に get_details イベントを送信し
                                                                            % #fitting_details{} を取得
                                                                            % riak_pipe_fitting に riak_pipe_vnode のプロセスを監視させる
                {ok, #fitting_details{q_limit=FQL}=Details} ->
                    erlang:monitor(process, Fitting#fitting.pid),           % fitting のプロセスを監視
                    
                    {ok, Pid} = riak_pipe_vnode_worker_sup:start_worker(    % worker を開始
                                  Sup, Details),
                                  
                    erlang:link(Pid),
                    Start = os:timestamp(),
                    Perf = #worker_perf{started=Start, last_time=Start},
                    ?T(Details, [worker], {vnode, {start, P}}),
                    
                    {ok, #worker{pid=Pid,
                                 fitting=Fitting,
                                 details=Details,
                                 state=init,
                                 inputs_done=false,
                                 q=queue:new(),
                                 q_limit=lists:min([WQL, FQL]),
                                 blocking=queue:new(),
                                 perf=Perf}};                               % #worker{} を返す
                                 
                gone ->
                    lager:error(
                      "Pipe worker startup failed:"
                      "fitting was gone before startup"),
                    worker_startup_failed
            end
            
        catch Type:Reason ->
                lager:error(
                  "Pipe worker startup failed:~n"
                  "   ~p:~p~n   ~p",
                  [Type, Reason, erlang:get_stacktrace()]),
                worker_startup_failed
        end.

riak_pipe_vnode_worker:init/1
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``riak_pipe_vnode_worker:init/1``::

    init([Partition, VnodePid, #fitting_details{module=Module}=FittingDetails]) ->
        try
            put(eunit, [{module, ?MODULE},
                        {partition, Partition},
                        {VnodePid, VnodePid},
                        {details, FittingDetails}]),
                        
            {ok, ModState} = Module:init(Partition, FittingDetails),    % fitting に指定されていたモジュールを初期化
            
            {ok, initial_input_request,
             #state{partition=Partition,
                    details=FittingDetails,
                    vnode=VnodePid,
                    modstate=ModState},
             0}                                                         % initial_input_request へ遷移
        catch Type:Error ->
                {stop, {init_failed, Type, Error}}
        end.

%% summary
        
参考資料
========

* Riak Pipe - Riak's Distributed Processing Framework - Bryan Fink, RICON2012

    - http://vimeo.com/53910999#at=0
    - http://hobbyist.data.riakcs.net:8080/ricon-riak-pipe.pdf
