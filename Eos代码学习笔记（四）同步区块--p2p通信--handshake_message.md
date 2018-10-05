上一篇整体上学习了net_plugin插件，但是具体的内容没有写。从这篇笔记开始，重点分析各节点之间是如何通过p2p插件同步区块的。首先，启动一个节点之后，当前节点向远程节点发送的第一个数据包就是handshake_message类型的（time时间戳类型除外），所以这篇笔记中分析handshake_message类型数据包的发送过程和接收过程。
**运行环境**：CLion编译器，并配置连接到主网节点。

####1、handshake_message消息结构
```
struct handshake_message {
      uint16_t                   network_version = 0; ///< incremental value above a computed base
      chain_id_type              chain_id; ///< used to identify chain
      fc::sha256                 node_id; ///< used to identify peers and prevent self-connect
      chain::public_key_type     key; ///< authentication key; may be a producer or peer key, or empty
      tstamp                     time;
      fc::sha256                 token; ///< digest of time to prove we own the private key of the key above
      chain::signature_type      sig; ///< signature for the digest
      string                     p2p_address;
      uint32_t                   last_irreversible_block_num = 0;
      block_id_type              last_irreversible_block_id;
      uint32_t                   head_num = 0;
      block_id_type              head_id;
      string                     os;
      string                     agent;
      int16_t                    generation;
   };
```
握手包内容包括：网络版本、chain_id、node_id、p2p_address、节点名称等配置信息，以及链的状态（当前节点的不可逆区块数、不可逆区块id、最新区块id、最新区块数）。其中区块数是指区块编号（1、2、3......），区块id是指32位的hash值。
####2、发送handshake_message消息
启动本地节点之后，会连接到配置文件中的p2p节点，并获取当前链信息、配置信息，然后构建handshake_message包并发送到其他p2p节点。
```
   void connection::send_handshake( ) {
      handshake_initializer::populate(last_handshake_sent); //填充消息内容
      last_handshake_sent.generation = ++sent_handshake_count;
      fc_dlog(logger, "Sending handshake generation ${g} to ${ep}",
              ("g",last_handshake_sent.generation)("ep", peer_name()));
      enqueue(last_handshake_sent);  //构建完成握手包之后，放入队列中
   }
```
发送的数据包内容如下：
```
//大小 348个字节  连接之后发送的报文
0040         5c 01 00 00 00 b6 04 ac a3 76 f2 06 b8 fc   2Ñ\....¶.¬£vò.¸ü
0050   25 a6 ed 44 db dc 66 54 7c 36 c6 c3 3e 3a 11 9f   %¦íDÛÜfT|6ÆÃ>:..
0060   fb ea ef 94 36 42 f0 e9 06 da b2 ea 2d 82 ce 71   ûêï.6Bðé.Ú²ê-.Îq
0070   45 c4 74 df 2f 3f 5f d9 df 11 af 8c 70 62 06 7a   EÄtß/?_Ùß.¯.pb.z
0080   5d de 3e 6b 87 10 22 18 0c 00 00 00 00 00 00 00   ]Þ>k..".........
0090   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
00a0   00 00 00 00 00 00 00 00 00 00 00 aa 8b b8 81 00   ...........ª.¸..
00b0   77 05 00 00 00 00 00 00 00 00 00 00 00 00 00 00   w...............
00c0   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
00d0   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
00e0   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
00f0   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0100   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0110   00 00 00 00 00 2d 6d 6f 6f 6e 69 6e 77 61 74 65   .....-mooninwate
0120   72 64 65 4d 61 63 42 6f 6f 6b 2d 50 72 6f 2e 6c   rdeMacBook-Pro.l
0130   6f 63 61 6c 3a 39 38 37 36 20 2d 20 64 61 62 32   ocal:9876 - dab2
0140   65 61 32 00 00 00 00 00 00 00 00 00 00 00 00 00   ea2.............
0150   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0160   00 00 00 00 00 00 00 01 00 00 00 00 00 00 01 40   ...............@
0170   51 47 47 7a b2 f5 f5 1c da 42 7b 63 81 91 c6 6d   QGGz²õõ.ÚB{c..Æm
0180   2c 59 aa 39 2d 5c 2c 98 07 6c b0 03 6f 73 78 10   ,Yª9-\,..l°.osx.
0190   22 45 4f 53 20 54 65 73 74 20 41 67 65 6e 74 22   "EOS Test Agent"
01a0   01 00  
```
####3、接收handshake_message消息
eos接收到其他节点发送过来的消息后，根据消息类型的不同，重载了不同的handle_message函数，其中handshake_message类型，主要功能包括两方面：
（1）、验证接收到的handshake包内容中的链id、节点id、网络版本等信息。
（2）、对比接收到的消息中链的状态和自身链的状态对比，然后进行区块同步。
```
//重载后的函数， 处理handshake_message消息
void net_plugin_impl::handle_message( connection_ptr c, const handshake_message &msg) {
      peer_ilog(c, "received handshake_message");
      if (!is_valid(msg)) {
         peer_elog( c, "bad handshake message");
         c->enqueue( go_away_message( fatal_other ));
         return;
      }
      controller& cc = chain_plug->chain();
      uint32_t lib_num = cc.last_irreversible_block_num( );
      uint32_t peer_lib = msg.last_irreversible_block_num;
      if (msg.generation == 1) {
         if( c->peer_addr.empty() || c->last_handshake_recv.node_id == fc::sha256()) {
            fc_dlog(logger, "checking for duplicate" );
            //遍历所有连接的节点，c代表当前连接中的节点 保证同一个p2p节点只存在一个连接
            ········
           }
         }
         if( msg.chain_id != chain_id) {
            elog( "Peer on a different chain. Closing connection");
            c->enqueue( go_away_message(go_away_reason::wrong_chain) );
            return;
         }
          ······
         if(  c->node_id != msg.node_id) {
            c->node_id = msg.node_id;
         }
        ········
      c->last_handshake_recv = msg;
      c->_logger_variant.reset();
      sync_master->recv_handshake(c,msg); //根据链的状态进行同步管理
   }
```
主要功能在于`recv_handshake`函数中同步区块管理。
```
void sync_manager::recv_handshake (connection_ptr c, const handshake_message &msg) {
      controller& cc = chain_plug->chain();
      uint32_t lib_num = cc.last_irreversible_block_num( );
      uint32_t peer_lib = msg.last_irreversible_block_num;
      reset_lib_num(c);
      c->syncing = false;

      //--------------------------------
      // sync need checks; (lib == last irreversible block)
      //
      // 0. my head block id == peer head id means we are all caugnt up block wise
      // 1. my head block num < peer lib - start sync locally
      // 2. my lib > peer head num - send an last_irr_catch_up notice if not the first generation
      //
      // 3  my head block num <= peer head block num - update sync state and send a catchup request
      // 4  my head block num > peer block num ssend a notice catchup if this is not the first generation
      //
      //-----------------------------

      uint32_t head = cc.fork_db_head_block_num( );
      block_id_type head_id = cc.fork_db_head_block_id();
      if (head_id == msg.head_id) {
         fc_dlog(logger, "sync check state 0");
         // notify peer of our pending transactions
         notice_message note;
         note.known_blocks.mode = none;
         note.known_trx.mode = catch_up;
         note.known_trx.pending = my_impl->local_txns.size();
         c->enqueue( note );
         return;
      }
      if (head < peer_lib) {
         fc_dlog(logger, "sync check state 1");
         // wait for receipt of a notice message before initiating sync
         if (c->protocol_version < proto_explicit_sync) {
            start_sync( c, peer_lib);
         }
         return;
      }
      if (lib_num > msg.head_num ) {
         fc_dlog(logger, "sync check state 2");
         if (msg.generation > 1 || c->protocol_version > proto_base) {
            notice_message note;
            note.known_trx.pending = lib_num;
            note.known_trx.mode = last_irr_catch_up;
            note.known_blocks.mode = last_irr_catch_up;
            note.known_blocks.pending = head;
            c->enqueue( note );
         }
         c->syncing = true;
         return;
      }

      if (head <= msg.head_num ) {
         fc_dlog(logger, "sync check state 3");
         verify_catchup (c, msg.head_num, msg.head_id);
         return;
      }
      else {
         fc_dlog(logger, "sync check state 4");
         if (msg.generation > 1 ||  c->protocol_version > proto_base) {
            notice_message note;
            note.known_trx.mode = none;
            note.known_blocks.mode = catch_up;
            note.known_blocks.pending = head;
            note.known_blocks.ids.push_back(head_id);
            c->enqueue( note );
         }
         c->syncing = true;
         return;
      }
      elog ("sync check failed to resolve status");
   }
```
这里的对比区块信息，主要分为五种情况：
（1）、接收到的最新区块id与自身最新区块id相同。
（2）、自身最新区块数小于接收到的区块的不可逆数。（自己的最新区块高度比远程节点的不可逆区块高度还要低，需要同步。）
（3）、自身不可逆区块数大于接收到的最新区块数。（与（2）相反，通知远程节点，需要同步。发送的是notice_message类型的消息，下一篇笔记中再写。）
（4）、自身最新区块数小于接收到的最新区块数。（需要同步，发送的是request_message消息，后面再写这种消息类型。）
（5）、自身最新区块数大于接收到的最新区块数。（与（4）相反，告通知远程节点，需要同步。发送的是notice_message类型的消息，下一篇笔记中在写。）
以上便是handshake_message消息的发送与接收过程。此篇笔记写到本地节点将自己的配置信息、链的状态告诉远程节点，远程节点接收到这些信息之后，发现自身链比本地节点的链更长，所以给本地节点发送消息同步区块。（消息类型为notice_message，下一篇分析本地节点接收到notice_message类型的消息之后，如何同步区块）。