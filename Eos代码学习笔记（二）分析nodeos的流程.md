####本篇笔记主要分析nodeos程序的流程
1、代码所在路径（yourpath换成你的路径）
`yourpath/eos/programs/nodeos/main.cpp`
2、首先main函数中的代码并不长，可以通过五个类来学习这段代码。
```
      app().set_version(eosio::nodeos::config::version);
      app().register_plugin<history_plugin>();

      auto root = fc::app_path();
      app().set_default_data_dir(root / "eosio/nodeos/data" );
      app().set_default_config_dir(root / "eosio/nodeos/config" );
      http_plugin::set_defaults({
         .address_config_prefix = "",
         .default_unix_socket_path = "",
         .default_http_port = 8888
      });
      if(!app().initialize<chain_plugin, http_plugin, net_plugin, producer_plugin>(argc, argv))
         return INITIALIZE_FAIL;
      initialize_logging();
      ilog("nodeos version ${ver}", ("ver", app().version_string()));
      ilog("eosio root is ${root}", ("root", root.string()));
      app().startup();
      app().exec();
```
上面的代码，均是通过调用app()的成员函数来实现的。我们很明显的看到main函数的生命周期和app()返回的对象是完全相同的。所以，第一个分析的就是app()返回的类对象。
####1、application类（单例模式）
查看app()函数，发现其返回的是一个application类对象的一个引用，对象是通过application类的静态方法instance创建的。instance方法创建一个application类对象，并返回其引用。由此实现一个单例模式，每次调用app()仅创建一个对象。
`application& app() { return application::instance(); }`
`application& application::instance() {
   static application _app;
   return _app;
}`
main函数中出现的`set_version`和`set_default_data_dir`和`set_default_config_dir`均为简单的成员函数，这里略过不写。
主要分析一下其他几个函数：`register_plugin `、`initialize `、`startup `、`exec`。
**register_plugin函数**是一个模板函数，功能是对插件进行注册。注册即创建一个插件对象，并保存在application对象的plugins变量里面。注册插件的时候是需要注册其依赖插件的。比如p2p插件依赖于chain插件，那么在注册p2p插件时，也需要注册chain插件。其中获取请求插件，采用了一个宏实现了递归获取的方式，后续在plugins类中详细说明。
```      
 template<typename Plugin>
    auto& register_plugin() {
    auto existing = find_plugin<Plugin>();   
    if(existing)
         return *existing;
    auto plug = new Plugin();
    plugins[plug->name()].reset(plug);
    plug->register_dependencies(); // 比较重要的地方，调用的是plugins类中的函数，后续分析。
    return *plug;
 }
```
**initialize函数**是一个变参模板函数，功能是对插件进行初始化（调用每个插件的initialize方法）。内部将变参参数初始化为一个vector向量，并调用initialize_impl函数来具体实现。
```
  //application.hpp
  template<typename... Plugin>
     bool                 initialize(int argc, char** argv) {
     return initialize_impl(argc, argv, {find_plugin<Plugin>()...});
  }
 //application.cpp  调用插件的initialize方法。
  for (auto plugin : autostart_plugins)
       if (plugin != nullptr && plugin->get_state() == abstract_plugin::registered)
           plugin->initialize(options);
```
**startup函数**很简单，功能用来启动初始化过的插件，内部调用每个插件的startup函数来启动。
`for (auto plugin : initialized_plugins)
         plugin->startup();`
**exec函数**使用了boost::asio::io_service io服务，在每个启动的插件线程里使用异步io的地方，均使用的是application对象的io服务，exec函数创建了异步IO异常终止的信号，并对其做了资源释放处理。
```
void application::exec() {
   std::shared_ptr<boost::asio::signal_set> sigint_set(new boost::asio::signal_set(*io_serv, SIGINT));
   sigint_set->async_wait([sigint_set,this](const boost::system::error_code& err, int num) {
     quit();
     sigint_set->cancel();
   });

   std::shared_ptr<boost::asio::signal_set> sigterm_set(new boost::asio::signal_set(*io_serv, SIGTERM));
   sigterm_set->async_wait([sigterm_set,this](const boost::system::error_code& err, int num) {
     quit();
     sigterm_set->cancel();
   });

   std::shared_ptr<boost::asio::signal_set> sigpipe_set(new boost::asio::signal_set(*io_serv, SIGPIPE));
   sigpipe_set->async_wait([sigpipe_set,this](const boost::system::error_code& err, int num) {
     quit();
     sigpipe_set->cancel();
   });

   io_serv->run();

   shutdown(); /// perform synchronous shutdown
}
```
####2、abstract_plugin类
通过对application类的几个函数的分析，发现初始化启动插件真正的实现，是通过插件自身的initialize方法和startup方法来实现的，下面主要分析其他四个类--插件类。这四个类可以用一个图来表示：
![插件类示意图](https://upload-images.jianshu.io/upload_images/10551960-4d658988b1aee61b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中abstract_plugin是虚基类，plugin是它的子类，http_plugin是plugin的子类，http_plugin_impl包含在http_plugin中。即前三个类是继承关系，后面一个类是包含关系。
回到上面遗留的问题--“插件是如何注册其依赖插件的”。
`plug->register_dependencies();` 
父类指针指向的是子类对象，该方法在plugin类中实现。
```
  virtual void register_dependencies() {
       static_cast<Impl*>(this)->plugin_requires([&](auto& plug){});
 }
```
plugin_requires函数并非Plugin类的成员函数，是其子类的成员函数，所以父类指针需要转换为子类指针，才可以调用。该函数由一个宏来实现，并实现了递归调用。(BOOST_PP_SEQ_FOR_EACH宏的作用，将第三个参数序列分别与第二个参数进行按照第一个参数的方式进行拼接，)
`APPBASE_PLUGIN_REQUIRES((chain_plugin))`
```
//宏代码：
#define APPBASE_PLUGIN_REQUIRES_VISIT( r, visitor, elem ) \
  visitor( appbase::app().register_plugin<elem>() ); 

#define APPBASE_PLUGIN_REQUIRES( PLUGINS )                               \
   template<typename Lambda>                                           \
   void plugin_requires( Lambda&& l ) {                                \
      BOOST_PP_SEQ_FOR_EACH( APPBASE_PLUGIN_REQUIRES_VISIT, l, PLUGINS ) \
   }
//-----------------------------------------展开结果-------------------------------------------
//宏展开：
template<typename Lambda>                                           
   void plugin_requires( Lambda&& l ) {                                
      BOOST_PP_SEQ_FOR_EACH( APPBASE_PLUGIN_REQUIRES_VISIT, l, (chain_plugin) ) 
   }
//继续展开
template<typename Lambda>                                           
   void plugin_requires( Lambda&& l ) {                                
      l( appbase::app().register_plugin<chain_plugin>() );
   }
```
所以plugin_requires函数为http_plugin类的成员函数，参数为一个lambda表达式。调用过程为`static_cast<Impl*>(this)->plugin_requires([&](auto& plug){});`，传入的表达式为[&](auto& plug){}。
所以调用此表达式，参数为appbase::app().register_plugin<chain_plugin>()。使用宏的方式实现成员函数的递归调用，之前没有遇到过这种写法，很奇怪。
再来看abstract_plugin类，此类很简单，代码很少，实现了插件的初始化、启动、停止等几个虚方法，是一个虚基类。具体代码均有其派生类的多态实现，所以下面分析另外三个类。
```
     virtual void set_program_options( options_description& cli, options_description& cfg ) = 0;
     virtual void initialize(const variables_map& options) = 0;
     virtual void startup() = 0;
     virtual void shutdown() = 0;
```
####3、plugin类
plugin类是eos各个插件的父类，保存了插件的状态（注册、启动等），实现了初始化、启动等接口，内部逻辑是通过子类来实现的。
```
  virtual void register_dependencies() {
            static_cast<Impl*>(this)->plugin_requires([&](auto& plug){});
         }

         virtual void initialize(const variables_map& options) override {
            if(_state == registered) {
               _state = initialized;
               static_cast<Impl*>(this)->plugin_requires([&](auto& plug){ plug.initialize(options); });
               static_cast<Impl*>(this)->plugin_initialize(options);
               //ilog( "initializing plugin ${name}", ("name",name()) );
               app().plugin_initialized(*this);
            }
            assert(_state == initialized); /// if initial state was not registered, final state cannot be initiaized
         }

         virtual void startup() override {
            if(_state == initialized) {
               _state = started;
               static_cast<Impl*>(this)->plugin_requires([&](auto& plug){ plug.startup(); });
               static_cast<Impl*>(this)->plugin_startup();
               app().plugin_started(*this);
            }
            assert(_state == started); // if initial state was not initialized, final state cannot be started
         }
```
####4、net_plugin类（同级插件：http_plugin、chain_plugin等）
net_plugin类是plugin类的子类，该类主要实现了p2p插件的插件配置、初始化、启动、停止、广播区块的方法，但其内部实现均是通过的net_plugin_impl类的方法来完成的，net_plugin类包含一个net_plugin_impl类的实例（每个插件都是类似的结构）。
```
   class net_plugin : public appbase::plugin<net_plugin>
   {
      public:
        net_plugin();
        virtual ~net_plugin();

        APPBASE_PLUGIN_REQUIRES((chain_plugin))
        virtual void set_program_options(options_description& cli, options_description& cfg) override;

        void plugin_initialize(const variables_map& options);
        void plugin_startup();
        void plugin_shutdown();

        void   broadcast_block(const chain::signed_block &sb);

        string                       connect( const string& endpoint );
        string                       disconnect( const string& endpoint );
        optional<connection_status>  status( const string& endpoint )const;
        vector<connection_status>    connections()const;

        size_t num_peers() const;
      private:
        std::unique_ptr<class net_plugin_impl> my; 
   };
```
plugin_initialize 初始化插件，其本质是实例化net_plugin_impl。my为net_plugin_impl类对象的指针。
```
  void net_plugin::plugin_initialize( const variables_map& options ) {
      ilog("Initialize net plugin");
      try {
         // 读取配置信息，初始化net_plugin_imul 对象的成员变量   
         peer_log_format = options.at( "peer-log-format" ).as<string>();
     
         my->network_version_match = options.at( "network-version-match" ).as<bool>();

         my->sync_master.reset( new sync_manager( options.at( "sync-fetch-span" ).as<uint32_t>()));
         my->dispatcher.reset( new dispatch_manager );
```
plugin_startup函数启动了一个p2p节点网络。包括1、设置监听循环，对其他节点发送过来的消息进行响应。2、根据配置文件中的seed节点信息，连接到其他节点，并发送消息请求，同步区块等消息（详细笔记写在下一篇）。通信用到的消息类型共分为如下几种：
```
  using net_message = static_variant<handshake_message,
                 chain_size_message,
                 go_away_message,
                 time_message,
                 notice_message,
                 request_message,
                 sync_request_message,
                 signed_block,
                 packed_transaction>;
```
####5、net_plugin_impl类
该类涉及到的是核心业务的具体实现。也是最复杂的一个类，下一篇笔记重点写下net插件的学习过程。
net_plugin_impl类通信使用的是boost::asio异步通信的库。该类成员变量包括了当前连接的p2p节点对象、区块同步管理对象、链id、节点id、网络通信用的相关变量。
```
   class net_plugin_impl {
   public:
      unique_ptr<tcp::acceptor>        acceptor;
      tcp::endpoint                    listen_endpoint;
      string                           p2p_address;
      uint32_t                         max_client_count = 0;
      uint32_t                         max_nodes_per_host = 1;
      uint32_t                         num_clients = 0;

      vector<string>                   supplied_peers;
      vector<chain::public_key_type>   allowed_peers; ///< peer keys allowed to connect
      std::map<chain::public_key_type,
               chain::private_key_type> private_keys; ///< overlapping with producer keys, also authenticating non-producing nodes

      enum possible_connections : char {
         None = 0,
            Producers = 1 << 0,
            Specified = 1 << 1,
            Any = 1 << 2
            };
      possible_connections             allowed_connections{None};

      connection_ptr find_connection( string host )const;

      std::set< connection_ptr >       connections;               // 已连接的p2p seed节点 指针集合
      bool                             done = false;
      unique_ptr< sync_manager >       sync_master;               // 区块同步管理类指针
      unique_ptr< dispatch_manager >   dispatcher;
      .......
}
```
net_plugin插件的详细内容，下一篇笔记详细写。