上篇笔记写到远程节点给本地节点发送notice消息，通知本地节点同步区块。本篇笔记继续学习本地节点如何从远程节点同步区块，这次重点写notice消息的发送与处理过程。

####1、notice_message消息结构
```
//消息结构 定义部分
struct notice_message {
  notice_message () : known_trx(), known_blocks() {}
  ordered_txn_ids known_trx;
  ordered_blk_ids known_blocks;
};
//选择查看定义，跳转到
using ordered_txn_ids = select_ids<transaction_id_type>;
using ordered_blk_ids = select_ids<block_id_type>;
//选择查看定义，跳转到
template<typename T>
struct select_ids {
  select_ids () : mode(none),pending(0),ids() {}
  id_list_modes  mode;
  uint32_t       pending;
  vector<T>      ids;        //模板类
  bool           empty () const { return (mode == none || ids.empty()); }
};
```
notice_message消息类型包括id_list_modes类型的变量，该类型为一个枚举类型，包括`  enum id_list_modes {
    none,
    catch_up,
    last_irr_catch_up,
    normal
  };`，pending表示区块数目。
####2、发送notice_message消息
远程节点给本地节点发送notice_message，通知本地节点同步到不可逆区块。
```
//远程节点给本地节点发送的通知消息内容 消息调用部分
if (lib_num > msg.head_num ) {
   fc_dlog(logger, "sync check state 2");
   if (msg.generation > 1 || c->protocol_version > proto_base) {
      notice_message note;
      note.known_trx.pending = lib_num;
      note.known_trx.mode = last_irr_catch_up;   //类型为不可逆区块类型
      note.known_blocks.mode = last_irr_catch_up;
      note.known_blocks.pending = head;
      c->enqueue( note );
   }
   c->syncing = true;
   return;
}
```
远程节点发送的notice_message消息内容比较简单，包括不可逆区块数、最新区块数、同步方式（同步到不可逆区块或者同步到最新区块），然后放入到消息队列等待发送出去。
####3、接收notice_message消息
同上篇笔记所写的一样，处理notice_message消息同样在一个handle_message的重载函数里进行，参数为当前通信的远程节点的connect对象指针和消息。
```
void net_plugin_impl::handle_message( connection_ptr c, const notice_message &msg) {
      // peer tells us about one or more blocks or txns. When done syncing, forward on
      // notices of previously unknown blocks or txns,
      //
      peer_ilog(c, "received notice_message");
      c->connecting = false;
      request_message req;
      bool send_req = false;
      if (msg.known_trx.mode != none) {
         fc_dlog(logger,"this is a ${m} notice with ${n} blocks", ("m",modes_str(msg.known_trx.mode))("n",msg.known_trx.pending));
      }
      switch (msg.known_trx.mode) {
      case none:
         break;
      case last_irr_catch_up: {
         c->last_handshake_recv.head_num = msg.known_trx.pending;
         req.req_trx.mode = none;
         break;
      }
      case catch_up : {
         if( msg.known_trx.pending > 0) {
            // plan to get all except what we already know about.
            req.req_trx.mode = catch_up;
            send_req = true;
            size_t known_sum = local_txns.size();
            if( known_sum ) {
               for( const auto& t : local_txns.get<by_id>( ) ) {
                  req.req_trx.ids.push_back( t.id );
               }
            }
         }
         break;
      }
      case normal: {
         dispatcher->recv_notice (c, msg, false);
      }
      }

      if (msg.known_blocks.mode != none) {
         fc_dlog(logger,"this is a ${m} notice with ${n} blocks", ("m",modes_str(msg.known_blocks.mode))("n",msg.known_blocks.pending));
      }
      switch (msg.known_blocks.mode) {
      case none : {
         if (msg.known_trx.mode != normal) {
            return;
         }
         break;
      }
      case last_irr_catch_up: //同步到最新区块和同步到不可逆区块 调用的函数相同
      case catch_up: {
         sync_master->recv_notice(c,msg); //通过sync_master进行区块同步
         break;
      }
      case normal : {
         dispatcher->recv_notice (c, msg, false);
         break;
      }
      default: {
         peer_elog(c, "bad notice_message : invalid known_blocks.mode ${m}",("m",static_cast<uint32_t>(msg.known_blocks.mode)));
      }
      }
      fc_dlog(logger, "send req = ${sr}", ("sr",send_req));
      if( send_req) {
         c->enqueue(req);
      }
   }
```
由于远程节点发送的notice_message数据包内容为同步到不可逆区块，mode为`last_irr_catch_up`，所以代码中调用`sync_master->recv_notice(c,msg)`函数来进行区块同步。此函数同步区块分为两种情况
```
   void sync_manager::recv_notice (connection_ptr c, const notice_message &msg) {
      fc_ilog (logger, "sync_manager got ${m} block notice",("m",modes_str(msg.known_blocks.mode)));
      if (msg.known_blocks.mode == catch_up) {
         if (msg.known_blocks.ids.size() == 0) {
            elog ("got a catch up with ids size = 0");
         }
         else {
             //同步到最新区块
            verify_catchup(c,  msg.known_blocks.pending, msg.known_blocks.ids.back());
         }
      }
      else {
         c->last_handshake_recv.last_irreversible_block_num = msg.known_trx.pending;
         reset_lib_num (c);
         //同步到不可逆区块
         start_sync(c, msg.known_blocks.pending);
      }
   }
```
我们重点看同步到不可逆区块，函数为`start_sync`，参数为connection对象指针和消息中包含的最新区块数（不过我觉得应该是不可逆区块数）。
```
   void sync_manager::start_sync( connection_ptr c, uint32_t target) {
      if( target > sync_known_lib_num) {
         sync_known_lib_num = target;
      }

      if (!sync_required()) {
         uint32_t bnum = chain_plug->chain().last_irreversible_block_num();
         uint32_t hnum = chain_plug->chain().fork_db_head_block_num();
         fc_dlog( logger, "We are already caught up, my irr = ${b}, head = ${h}, target = ${t}",
                  ("b",bnum)("h",hnum)("t",target));
         return;
      }

      if (state == in_sync) {
         set_state(lib_catchup);
         sync_next_expected_num = chain_plug->chain().last_irreversible_block_num() + 1;
      }

      fc_ilog(logger, "Catching up with chain, our last req is ${cc}, theirs is ${t} peer ${p}",
              ( "cc",sync_last_requested_num)("t",target)("p",c->peer_name()));

      request_next_chunk(c);
   }
```
在这里，成员变量sync_known_lib_num赋值为sync_known_lib_num和target的最大值。该成员变量表示已知的当前不可逆区块数，可见上一步同步到不可逆区块的调用应该为`start_sync(c, msg.known_trx.pending);`，在这个函数里，主要是检查区块同步信息，然后调用`request_next_chunk(c)`函数进行区块同步。
```
   void sync_manager::request_next_chunk( connection_ptr conn ) {
      uint32_t head_block = chain_plug->chain().fork_db_head_block_num();
      ······ //省略代码
      if( sync_last_requested_num != sync_known_lib_num ) {
         uint32_t start = sync_next_expected_num;
         uint32_t end = start + sync_req_span - 1; // sync_req_span为配置文件中设置，每次同步多少个区块，默认是100个。
         if( end > sync_known_lib_num )
            end = sync_known_lib_num;
         if( end > 0 && end >= start ) {
            fc_ilog(logger, "requesting range ${s} to ${e}, from ${n}",
                    ("n",source->peer_name())("s",start)("e",end));
            source->request_sync_blocks(start, end);//同样是发送某种消息来同步区块
            sync_last_requested_num = end;
         }
      }
   }
```
查看request_next_chunk函数的功能，我们发现内部调用request_sync_blocks函数来发送同步消息，具体实现需要进入分析。
```
void connection::request_sync_blocks (uint32_t start, uint32_t end) {
      sync_request_message srm = {start,end};
      enqueue( net_message(srm));
      sync_wait();
   }
```
request_sync_blocks函数内部实现很简单，构造了一个sync_request_message类型的消息，然后放入到消息队列中，消息的内容为起始区块和结束区块，每次请求100个（config.ini中默认设置，可以修改）。
此时这篇笔记讨论的消息类型先到这里，下一篇笔记讨论sync_request_message类型的消息，分析本地节点是如何从远程同步区块的。