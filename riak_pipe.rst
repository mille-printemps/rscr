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
    ok = riak_pipe:queue_work(Pipe, "hello"),                        % riak_pipe:exec/2 から得た Pipe を渡して "hello" を送信
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
``riak_pipe_vnode:queue_work/4``::

    queue_work(#fitting{chashfun=follow}=Fitting,                    % Options を設定していない場合
               Input, Timeout, UsedPreflist) ->
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

        
参考資料
-------

* Riak Pipe - Riak's Distributed Processing Framework - Bryan Fink, RICON2012
    - http://vimeo.com/53910999#at=0
    - http://hobbyist.data.riakcs.net:8080/ricon-riak-pipe.pdf
