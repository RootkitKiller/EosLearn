这篇笔记写同步区块过程中，发送的最后两种消息类型--sync_request_message与signed_block。上篇笔记写到远程节点将链的信息（不可逆区块数、最新区块数、同步方式等）作为notice_message方式发送给本地节点，本地节点接收到notice_message消息，向远程节点发送sync_request_message消息同步区块。

####1、同步区块过程图示
![本地节点从远程节点同步不可逆区块](https://upload-images.jianshu.io/upload_images/10551960-37a8188ae70cf2ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
####2、sync_request_message消息接收处理
继续上篇笔记中的内容，远程节点接收到本地节点发送过来的sync_request_message消息之后，处理函数：
```
   void net_plugin_impl::handle_message( connection_ptr c, const sync_request_message &msg) {
      if( msg.end_block == 0) {   //结束区块数是否为0
         c->peer_requested.reset();
         c->flush_queues();
      } else {
         c->peer_requested = sync_state( msg.start_block,msg.end_block,msg.start_block-1);
         c->enqueue_sync_block();
      }
   }
```
本地节点发送的同步消息，默认为1-100的区块数，则处理函数进入`enqueue_sync_block`
```
   bool connection::enqueue_sync_block() {
      controller& cc = app().find_plugin<chain_plugin>()->chain();
      if (!peer_requested)
         return false;
      uint32_t num = ++peer_requested->last;
      bool trigger_send = num == peer_requested->start_block;
      if(peer_requested->last == peer_requested->end_block) {
         peer_requested.reset();
      }
      try {
         signed_block_ptr sb = cc.fetch_block_by_number(num);
         if(sb) {
            enqueue( *sb, trigger_send);
            return true;
         }
      } catch ( ... ) {
         wlog( "write loop exception" );
      }
      return false;
   }
```
num变量为peer_requested成员的last字段的值，表示当前需要同步的区块数。每次调用`enqueue_sync_block`函数，则该字段加一，即发送下一个区块给本地节点。然后调用`fetch_block_by_number`函数获取自己的第num个区块，并构造signed_block消息，然后放入到消息队列，这样就把本地节点请求的一个区块发送给了本地节点。
```
  boost::asio::async_write(*socket, bufs, [c](boost::system::error_code ec, std::size_t w) {
    try {
      ········//省略代码
        while (conn->out_queue.size() > 0) {
            conn->out_queue.pop_front();
        }
        conn->enqueue_sync_block();    ///同步下一个区块
        conn->do_queue_write();
      }
      ······ //省略代码
   }
```
在net_plugin插件处理消息队列的时候，会在异步发送消息的回调函数里，发送下一个区块给本地节点。
####3、接收signed_block消息
本地节点接收远程节点发送过来的signed_block消息，消息内容为区块详细数据
```
   void net_plugin_impl::handle_message( connection_ptr c, const signed_block &msg) {
      controller &cc = chain_plug->chain();
      block_id_type blk_id = msg.id();
      uint32_t blk_num = msg.block_num();
      fc_dlog(logger, "canceling wait on ${p}", ("p",c->peer_name()));
      c->cancel_wait();

      try {
         //查看本地是否存在该id的区块
         if( cc.fetch_block_by_id(blk_id)) {
            sync_master->recv_block(c, blk_id, blk_num);
            return;
         }
      } catch( ...) {
         // should this even be caught?
         elog("Caught an unknown exception trying to recall blockID");
      }

      dispatcher->recv_block(c, blk_id, blk_num);
      fc::microseconds age( fc::time_point::now() - msg.timestamp);
      peer_ilog(c, "received signed_block : #${n} block age in secs = ${age}",
              ("n",blk_num)("age",age.to_seconds()));

      go_away_reason reason = fatal_other;
      try {
        //写入区块数据，accept_block函数用到了boost库里面的信号插槽signal2库，这里没有分析清楚。
         signed_block_ptr sbp = std::make_shared<signed_block>(msg);
         chain_plug->accept_block(sbp); //, sync_master->is_active(c));
         reason = no_reason;
      } catch( const unlinkable_block_exception &ex) {
         peer_elog(c, "bad signed_block : ${m}", ("m",ex.what()));
         reason = unlinkable;
      } catch( const block_validate_exception &ex) {
         peer_elog(c, "bad signed_block : ${m}", ("m",ex.what()));
         elog( "block_validate_exception accept block #${n} syncing from ${p}",("n",blk_num)("p",c->peer_name()));
         reason = validation;
      } catch( const assert_exception &ex) {
         peer_elog(c, "bad signed_block : ${m}", ("m",ex.what()));
         elog( "unable to accept block on assert exception ${n} from ${p}",("n",ex.to_string())("p",c->peer_name()));
      } catch( const fc::exception &ex) {
         peer_elog(c, "bad signed_block : ${m}", ("m",ex.what()));
         elog( "accept_block threw a non-assert exception ${x} from ${p}",( "x",ex.to_string())("p",c->peer_name()));
         reason = no_reason;
      } catch( ...) {
         peer_elog(c, "bad signed_block : unknown exception");
         elog( "handle sync block caught something else from ${p}",("num",blk_num)("p",c->peer_name()));
      }

      update_block_num ubn(blk_num);
      if( reason == no_reason ) {
         for (const auto &recpt : msg.transactions) {
            auto id = (recpt.trx.which() == 0) ? recpt.trx.get<transaction_id_type>() : recpt.trx.get<packed_transaction>().id();
            auto ltx = local_txns.get<by_id>().find(id);
            if( ltx != local_txns.end()) {
               local_txns.modify( ltx, ubn );
            }
            auto ctx = c->trx_state.get<by_id>().find(id);
            if( ctx != c->trx_state.end()) {
               c->trx_state.modify( ctx, ubn );
            }
         }
         sync_master->recv_block(c, blk_id, blk_num);
      }
      else {
         sync_master->rejected_block(c, blk_num);
      }
   }
```
![signed_block 消息内容](https://upload-images.jianshu.io/upload_images/10551960-7ec6775be6315807.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
signed_block 区块报文内容
```
0040             b9 00 00 00 07 dc f9 5c 45 00 00 00 00 00   .F¹....Üù\E.....
0050   ea 30 55 00 00 00 00 00 01 40 51 47 47 7a b2 f5   ê0U......@QGGz²õ
0060   f5 1c da 42 7b 63 81 91 c6 6d 2c 59 aa 39 2d 5c   õ.ÚB{c..Æm,Yª9-\
0070   2c 98 07 6c b0 00 00 00 00 00 00 00 00 00 00 00   ,..l°...........
0080   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0090   00 00 00 00 00 e0 24 4d b4 c0 2d 68 ae 64 de c1   .....à$M´À-h®dÞÁ
00a0   60 31 0e 24 7b b0 4e 5c b5 99 af b7 c1 47 10 fb   `1.${°N\µ.¯·ÁG.û
00b0   f3 f4 57 6c 0e 00 00 00 00 00 00 00 20 60 15 f0   óôWl........ `.ð
00c0   39 e2 fd d0 df b2 31 6f ea 28 67 90 c6 b1 55 4f   9âýÐß²1oê(g.Æ±UO
00d0   28 5a 54 e7 d2 5d b3 ea 91 ef 2e 8c 11 0a 57 e2   (ZTçÒ]³ê.ï....Wâ
00e0   8e fb ff e0 92 c0 e0 2d 92 f2 88 5e 72 48 43 b4   .ûÿà.Àà-.ò.^rHC´
00f0   a5 5f 29 0e 20 9f 87 1f 41 bb 39 3c 84 00 00      ¥_). ...A»9<...:
```
