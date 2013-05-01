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
=====

riak_pipe_app.erl
-----------------

- application

``riak_pipe_app:start/2``::

    start(_StartType, _StartArgs) ->
        %% startup mostly copied from riak_kv
        catch cluster_info:register_app(riak_pipe_cinfo),

        case riak_pipe_sup:start_link() of                 % riak_pipe の supervisor を開始
            {ok, Pid} ->
                riak_core:register(riak_pipe, [
                 {vnode_module, riak_pipe_vnode},
                 {stat_mod, riak_pipe_stat}
                ]),
                {ok, Pid};
            {error, Reason} ->
                {error, Reason}
        end.

        
riak_pipe_sup.erl
-----------------

- supervisor

``riak_pipe_sup:init/1``::

    init([]) ->
        %% ordsets = enabled traces are represented as ordsets in fitting_details
        %% sets = '' sets ''
        riak_core_capability:register(
          {riak_pipe, trace_format}, [ordsets, sets], sets),

        VMaster = {riak_pipe_vnode_master,                                   % riak_core_vnode_master を開始
                   {riak_core_vnode_master, start_link, [riak_pipe_vnode]},
                   permanent, 5000, worker, [riak_core_vnode_master]},
                   
        BSup = {riak_pipe_builder_sup,                                       % riak_pipe_builder の supervisor を開始
                {riak_pipe_builder_sup, start_link, []},                     % restart strategy が simple_one_for_one
                   permanent, 5000, supervisor, [riak_pipe_builder_sup]},    % この時点では riak_pipe_builder は開始されない
                   
        FSup = {riak_pipe_fitting_sup,                                       % riak_pipe_fitting の supervisor を開始
                {riak_pipe_fitting_sup, start_link, []},                     % restart strategy が simple_one_for_one
                permanent, 5000, supervisor, [riak_pipe_fitting_sup]},       % この時点では riak_pipe_fitting は開始されない
                
        CSup = {riak_pipe_qcover_sup,                                        % riak_qcover の supervisor を開始
                {riak_pipe_qcover_sup, start_link, []},                      % restart strategy が simple_one_for_one
                permanent, 5000, supervisor, [riak_pipe_qcover_sup]},        % この時点では riak_pipe_qcover は開始されない
                
        {ok, { {one_for_one, 5, 10}, [VMaster, BSup, FSup, CSup]} }.         % one_for_one, 仮に子プロセスが落ちるとそのプロセスのみ再び開始される

        
クライアント
==========

- riak_pipe.erl にクライアントの API が定義されている。
    - pipeline の構築
    - 入力の送信


pipeline の構築
--------------

riak_pipe.erl

- riak_pipe_builder を使って pipeline を構築する。
    - riak_pipe_builder を開始
    - riak_pipe_fitting を開始
    - riak_pipe_builder と riak_pipe_fitting は子プロセスとして動的に追加される
        - supervisor が落ちて再開されても子プロセスは自動的に再開されない
        - riak_pipe_builder と riak_pipe_fitting がお互いに erlang:monitor する実装になっている
- #pipe{} を返す


``riak_pipe:exec/2``::

    exec(Spec, Options) ->
        [ riak_pipe_fitting:validate_fitting(F) || F <- Spec ],
        CorrectOptions = correct_trace(
                           validate_sink_type(
                             ensure_sink(Options))),               % Options が [] であった場合は生成される
                                                                   % その場合 [{sink, #fitting{pid=self(), ref=make_ref(), chashfun=sink}}]
                                                                   % となるので、Sink はクライアントプロセスになる?
    riak_pipe_builder_sup:new_pipeline(Spec, CorrectOptions).      % pipeline を構築

    
riak_pipe_builder_sup.erl

- riak_pipe_builder の supervisor

``riak_pipe_builder_sup:new_pipeline/2``::

    new_pipeline(Spec, Options) ->
        case supervisor:start_child(?MODULE, [Spec, Options]) of    % riak_pipe_builder を開始 -> supervisor の子プロセスとして追加
            {ok, Pid, Ref} ->
                case riak_pipe_builder:pipeline(Pid) of             % ``pipeline`` イベントを送信       
                    {ok, #pipe{sink=#fitting{ref=Ref}}=Pipe} ->
                        riak_pipe_stat:update({create, Pid}),       % 統計情報を収集
                        {ok, Pipe};                                 % #pipe{} を返す -> exec の返り値
                    _ ->
                        riak_pipe_stat:update(create_error),
                        {error, startup_failure}
                end;
            Error ->
                riak_pipe_stat:update(create_error),
                Error
        end.

        
riak_pipe_builder.erl

- gen_fsm

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
                    {fittings, Fittings}]),                                % pipe の情報を process dictionary へ格納 -> unit test に使う?
        {ok, wait_pipeline_shutdown,
        #state{options=Options,
                pipe=Pipe,
                alive=Fittings,
                sinkmon=SinkMon}}.                                         % ``wait_pipeline_shutdown`` へ遷移

                
``riak_pipe_builder:start_fittings/2``::

    start_fittings(Spec, Options) ->
        [Tail|Rest] = lists:reverse(Spec),                                 % Spec のリストを反転
        ClientOutput = client_output(Options),
        lists:foldl(fun(FitSpec, [{Output,_}|_]=Acc) ->
                            [start_fitting(FitSpec, Output, Options)|Acc]
                    end,
                    [start_fitting(Tail, ClientOutput, Options)],
                    Rest).                                                 % 反転した Spec のリストに順に start_fitting/3 を適用
                                                                           % [#fitting{pid, ref, chashfun, nval}, Ref}, ...] を返す

``riak_pipe_builder:start_fitting/3``::
 
    start_fitting(Spec, Output, Options) ->
        ?DPF("Starting fitting for ~p", [Spec]),
        {ok, Pid, Fitting} = riak_pipe_fitting_sup:add_fitting(
                               self(), Spec, Output, Options),             % riak_pipe_fitting を開始
        Ref = erlang:monitor(process, Pid),                                % riak_pipe_fitting を監視
        {Fitting, Ref}.                                                    % {#fitting{pid, ref, chashfun, nval}, Ref} を返す
                                                                           % pid は fitting の pid, ref は Sink の ref

        
riak_pipe_fitting_sup.erl

- riak_pipe_fitting の supervisor
        
``riak_pipe_fitting_sup:add_fitting/4``::

    add_fitting(Builder, Spec, Output, Options) ->
        ?DPF("Adding fitting for ~p", [Spec]),
        supervisor:start_child(?SERVER, [Builder, Spec, Output, Options]). % riak_pipe_fitting を開始 -> supervisor の子プロセスとして追加

        
riak_pipe_fitting.erl

- gen_fsm

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
                                   q_limit=QLimit},

        ?T(Details, [], {fitting, init_started}),                    % riak_pipe_log.hrl に定義されているマクロ
                                                                     % ``riak_pipe_log:trace/3`` を呼び出している
        erlang:monitor(process, Builder),                            % riak_pipe_builder を監視

        ?T(Details, [], {fitting, init_finished}),

        put(eunit, [{module, ?MODULE},
                    {fitting, Fitting},
                    {details, Details},
                    {builder, Builder}]),
        {ok, wait_upstream_eoi,
         #state{builder=Builder, details=Details, workers=[],
            ref=Output#fitting.ref}}.                                % ``wait_upstream_eoi`` へ遷移

            
入力の送信
---------

- クライアントは　``riak_pipe:queue_work/2`` を呼ぶ。
- ``riak_pipe:queue:work/3`` から最終的に ``riak_pipe_vnode:queue:work/4`` が呼ばれる。
- ``riak_pipe_vnoce:queue:work/4`` は fitting spec に設定される chashfun (consistent-hashing function) により4通り定義されている。

riak_pipe.erl

``riak_pipe:queue_work/3``::

    queue_work(#pipe{fittings=[{_,Head}|_]}, Input, Timeout)
      when Timeout =:= infinity; Timeout =:= noblock ->
        riak_pipe_vnode:queue_work(Head, Input, Timeout).            % 先頭の fitting (#fitting{}) を渡して　``riak_pipe_vnode:queue_work/3`` を呼ぶ


riak_pipe_vnode.erl

``riak_pipe:queue_work/4``::

    queue_work(#fitting{chashfun=follow}=Fitting,
               Input, Timeout, UsedPreflist) ->
        queue_work(Fitting, Input, Timeout, UsedPreflist, any_local_vnode());
    queue_work(#fitting{chashfun={Module, Function}}=Fitting,
           Input, Timeout, UsedPreflist) ->
    queue_work(Fitting, Input, Timeout, UsedPreflist,
               Module:Function(Input));
    queue_work(#fitting{chashfun=Hash}=Fitting,
           Input, Timeout, UsedPreflist) when not is_function(Hash) ->
            queue_work(Fitting, Input, Timeout, UsedPreflist, Hash);
    queue_work(#fitting{chashfun=HashFun}=Fitting,
           Input, Timeout, UsedPreflist) ->
        %% 1.0.x compatibility
        Hash = riak_pipe_fun:compat_apply(HashFun, [Input]),
        queue_work(Fitting, Input, Timeout, UsedPreflist, Hash).
            
参考資料
-------

- Riak Pipe - Riak's Distributed Processing Framework - Bryan Fink, RICON2012
    - http://vimeo.com/53910999#at=0
    - http://hobbyist.data.riakcs.net:8080/ricon-riak-pipe.pdf
