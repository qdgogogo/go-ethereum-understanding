# rlpx #

----------



rlpx是实际（非测试）连接使用的传输协议。其中应用到[迪菲-赫尔曼密钥交换](https://blog.csdn.net/fangxin205/article/details/54707633)。
这里面涉及到复杂的密码学计算，我也没有深入的了解，我知识对它所实现的功能做出了尝试性的解释。

	type rlpx struct {
		fd net.Conn		//通用的面向流的网络连接
	
		rmu, wmu sync.Mutex	//锁
		rw       *rlpxFrameRW
	}
	
	func newRLPX(fd net.Conn) transport {
		fd.SetDeadline(time.Now().Add(handshakeTimeout)) //5秒钟的截至时间
		return &rlpx{fd: fd}
	}
	
	func (t *rlpx) ReadMsg() (Msg, error) {
		t.rmu.Lock()
		defer t.rmu.Unlock()
		t.fd.SetReadDeadline(time.Now().Add(frameReadTimeout)) //读取完整消息所允许的最长时间（30秒）。这实际上是连接空闲的时间。
		return t.rw.ReadMsg()
	}
	
	func (t *rlpx) WriteMsg(msg Msg) error {
		t.wmu.Lock()
		defer t.wmu.Unlock()
		t.fd.SetWriteDeadline(time.Now().Add(frameWriteTimeout))//写入完整消息所允许的最长时间（20秒）
		return t.rw.WriteMsg(msg)
	}


rlpxFrameRW

rlpxFrameRW实现了RLPx框架的简化版本。 不支持分块消息，并且所有标头都等于zeroHeader。
  rlpxFrameRW对于多个goroutine的并发使用是不安全的。

	type rlpxFrameRW struct {
		conn io.ReadWriter
		enc  cipher.Stream
		dec  cipher.Stream
	
		macCipher  cipher.Block
		egressMAC  hash.Hash
		ingressMAC hash.Hash
	
		snappy bool
	}


1 doEncHandshake

doEncHandshake使用经过身份验证的消息运行协议握手。 协议握手是第一个经过身份验证的消息，还验证加密握手是否“正常”，远程端是否实际提供了正确的公钥。

链接的发起者被称为initiator。链接的被动接收者被成为receiver。


	func (t *rlpx) doEncHandshake(prv *ecdsa.PrivateKey, dial *ecdsa.PublicKey) (*ecdsa.PublicKey, error) {
		var (
			sec secrets  //secrets表示在加密握手期间协商的连接机密。
			err error
		)
		//生成了一个sec.可以理解为拿到了对称加密的密钥
		if dial == nil {
			sec, err = receiverEncHandshake(t.fd, prv)
		} else {
			sec, err = initiatorEncHandshake(t.fd, prv, dial)
		}
		if err != nil {
			return nil, err
		}
		t.wmu.Lock()
		t.rw = newRLPXFrameRW(t.fd, sec)
		t.wmu.Unlock()
		return sec.Remote.ExportECDSA(), nil
	}



1.1 initiatorEncHandshake

链接的发起者操作


	func initiatorEncHandshake(conn io.ReadWriter, prv *ecdsa.PrivateKey, remote *ecdsa.PublicKey) (s secrets, err error) {
		//创建encHandshake
		h := &encHandshake{initiator: true, remote: ecies.ImportECDSAPublic(remote)}
		authMsg, err := h.makeAuthMsg(prv)
		if err != nil {
			return s, err
		}
		//这个方法是一个组包方法，对msg进行rlp的编码。 填充一些数据。 然后使用对方的公钥把数据进行加密。 这意味着只有对方的私钥才能解密这段信息。
		authPacket, err := sealEIP8(authMsg, h)
		if err != nil {
			return s, err
		}
		if _, err = conn.Write(authPacket); err != nil { //4
			return s, err
		}
	
		authRespMsg := new(authRespV4)
		//读取对端的回应
		authRespPacket, err := readHandshakeMsg(authRespMsg, encAuthRespLen, prv, conn)
		if err != nil {
			return s, err
		}
		if err := h.handleAuthResp(authRespMsg); err != nil {
			return s, err
		}
		//调用secrets创建了共享秘密
		return h.secrets(authPacket, authRespPacket)
	}

1.1.1 makeAuthMsg

创建了发起人的handshake message

	func (h *encHandshake) makeAuthMsg(prv *ecdsa.PrivateKey) (*authMsgV4, error) {
		// Generate random initiator nonce.
		//生成随机发起人nonce
		h.initNonce = make([]byte, shaLen)
		_, err := rand.Read(h.initNonce)
		if err != nil {
			return nil, err
		}
		// Generate random keypair to for ECDH.
		////生成一个随机的私钥
		h.randomPrivKey, err = ecies.GenerateKey(rand.Reader, crypto.S256(), nil)
		if err != nil {
			return nil, err
		}
	
		// Sign known message: static-shared-secret ^ nonce
		// 以下的过程就是对消息签名的过程
		//返回静态共享密钥，即本地和远程静态节点密钥之间密钥协商的结果。
		
		token, err := h.staticSharedSecret(prv) // 1
		if err != nil {
			return nil, err
		}
		//将底数g与initNonce进行异或运算，然后将结果用私钥签名
		signed := xor(token, h.initNonce)  //2
		signature, err := crypto.Sign(signed, h.randomPrivKey.ExportECDSA()) //3
		if err != nil {
			return nil, err
		}
	
		msg := new(authMsgV4)
		copy(msg.Signature[:], signature)
		copy(msg.InitiatorPubkey[:], crypto.FromECDSAPub(&prv.PublicKey)[1:])
		copy(msg.Nonce[:], h.initNonce)
		msg.Version = 4
		return msg, nil
	}

1.1.2 readHandshakeMsg

这个方法会从两个地方调用。 一个是在initiatorEncHandshake。一个就是在receiverEncHandshake。 这个方法比较简单。 首先用一种格式尝试解码。如果不行就换另外一种。应该是一种兼容性的设置。 基本上就是使用自己的私钥进行解码然后调用rlp解码成结构体

	func readHandshakeMsg(msg plainDecoder, plainSize int, prv *ecdsa.PrivateKey, r io.Reader) ([]byte, error) {
		buf := make([]byte, plainSize)
		if _, err := io.ReadFull(r, buf); err != nil {
			return buf, err
		}
		// Attempt decoding pre-EIP-8 "plain" format.
		//尝试解码plain 这种形式的
		key := ecies.ImportECDSA(prv)
		if dec, err := key.Decrypt(buf, nil, nil); err == nil {
			msg.decodePlain(dec)
			return buf, nil
		}
		// 尝试EIP-8形式的
		prefix := buf[:2]
		size := binary.BigEndian.Uint16(prefix)
		if size < uint16(plainSize) {
			return buf, fmt.Errorf("size underflow, need at least %d bytes", plainSize)
		}
		buf = append(buf, make([]byte, size-uint16(plainSize)+2)...)
		if _, err := io.ReadFull(r, buf[plainSize:]); err != nil {
			return buf, err
		}
		dec, err := key.Decrypt(buf[2:], nil, prefix)
		if err != nil {
			return buf, err
		}
		// Can't use rlp.DecodeBytes here because it rejects
		// trailing data (forward-compatibility).
		s := rlp.NewStream(bytes.NewReader(dec), 0)
		return buf, s.Decode(msg)
	}

1.1.3 secrets

这个函数是在handshake完成之后调用。它通过自己的随机私钥和对端的公钥来生成一个共享秘密,这个共享秘密是瞬时的(只在当前这个链接中存在)。所以当有一天私钥被破解。 之前的消息还是安全的。

	func (h *encHandshake) secrets(auth, authResp []byte) (secrets, error) {
		ecdheSecret, err := h.randomPrivKey.GenerateShared(h.remoteRandomPub, sskLen, sskLen)
		if err != nil {
			return secrets{}, err
		}
	
		// 从短暂的密钥协议中获取基本秘密
		sharedSecret := crypto.Keccak256(ecdheSecret, crypto.Keccak256(h.respNonce, h.initNonce))
		aesSecret := crypto.Keccak256(ecdheSecret, sharedSecret)
		s := secrets{
			Remote: h.remote,
			AES:    aesSecret,
			MAC:    crypto.Keccak256(ecdheSecret, aesSecret),
		}
	
		// setup sha3 instances for the MACs
		mac1 := sha3.NewKeccak256()
		mac1.Write(xor(s.MAC, h.respNonce))
		mac1.Write(auth)
		mac2 := sha3.NewKeccak256()
		mac2.Write(xor(s.MAC, h.initNonce))
		mac2.Write(authResp)
		////收到的每个包都会检查其MAC值是否满足计算的结果。如果不满足说明有问题。
		if h.initiator {
			s.EgressMAC, s.IngressMAC = mac1, mac2
		} else {
			s.EgressMAC, s.IngressMAC = mac2, mac1
		}
	
		return s, nil
	}

1.2 receiverEncHandshake

它应该在连接的监听端调用。 prv 是本地客户端的私钥

	func receiverEncHandshake(conn io.ReadWriter, prv *ecdsa.PrivateKey) (s secrets, err error) {
		authMsg := new(authMsgV4)
		authPacket, err := readHandshakeMsg(authMsg, encAuthMsgLen, prv, conn)
		if err != nil {
			return s, err
		}
		h := new(encHandshake)
		if err := h.handleAuthMsg(authMsg, prv); err != nil {
			return s, err
		}
	
		authRespMsg, err := h.makeAuthResp()
		if err != nil {
			return s, err
		}
		var authRespPacket []byte
		if authMsg.gotPlain {
			authRespPacket, err = authRespMsg.sealPlain(h)
		} else {
			authRespPacket, err = sealEIP8(authRespMsg, h)
		}
		if err != nil {
			return s, err
		}
		if _, err = conn.Write(authRespPacket); err != nil {
			return s, err
		}
		return h.secrets(authPacket, authRespPacket)
	}

1.2.1 handleAuthMsg


	func (h *encHandshake) handleAuthMsg(msg *authMsgV4, prv *ecdsa.PrivateKey) error {
		// Import the remote identity.
		rpub, err := importPublicKey(msg.InitiatorPubkey[:])
		if err != nil {
			return err
		}
		h.initNonce = msg.Nonce[:]
		h.remote = rpub
	
		// Generate random keypair for ECDH.
		// If a private key is already set, use it instead of generating one (for testing).
		if h.randomPrivKey == nil {
			h.randomPrivKey, err = ecies.GenerateKey(rand.Reader, crypto.S256(), nil)
			if err != nil {
				return err
			}
		}
	
		// Check the signature.
		token, err := h.staticSharedSecret(prv) //5
		if err != nil {
			return err
		}
		signedMsg := xor(token, h.initNonce) //6
		remoteRandomPub, err := secp256k1.RecoverPubkey(signedMsg, msg.Signature[:])//6
		if err != nil {
			return err
		}
		h.remoteRandomPub, _ = importPublicKey(remoteRandomPub)
		return nil
	}

详解doEncHandshake加密握手，[原文在这里](https://blog.csdn.net/weixin_41814722/article/details/80680749):

	握手双方使用到的信息有:各自的公私钥地址对(iPrv,iPub,rPrv,rPub)、各自生成的随机公私钥对(iRandPrv,iRandPub,rRandPrv,rRandPub)、各自生成的临时随机数(initNonce,respNonce).
	其中i开头的表示发起方(initiator)信息,r开头的表示接收方(receiver)信息.

	1.1.1 中对应的数字标记
	握手开始:
	1.发起方(initiator)使用自己的私钥iPrv和对方的公钥rPub生成一个静态共享私密(token):
		token := iPrv*(rPub.X, rPub.Y)
	  可见token是由己方私钥和对方公钥扩展而成的椭圆曲线上的点做有限域标量乘积得到(与私钥产生公钥的过程类似).
	2.发起方生成一个随机数initNonce,和token异或生成一个待签名信息:
		unsigned := initNonce ⊕ token
	3.发起方用随机生成的私钥iRandPrv对待签名信息进行签名:
		signature := Sign(unsigned, iRandPrv)
	4.发起方将initNonce、signature、iPub打包成authMsg发送给接收方

	1.2.1 中对应的数字标记
	接收方收到握手请求后:
	5.接收方(receiver)用自己的私钥rPrv和发起方的公钥iPub生成一个token:
		token := rPrv*(iPub.X, iPub.Y)
	  由于公钥是由私钥产生的,有(iPub.X, iPub.Y) = iPrv*G(x0, y0),而G(x0, y0)是ECDSA椭圆曲线给定的初始点
	  从而接收方生成的token与发起方一致,它的值都是iPrv*rPrv*G(x0, y0).
	6.接收方用自己生成的token与发起方发送过来的initNonce异或得到签名前的信息unsigned,用unsigned和signature可以导出发送方的随机公钥iRandPub.
	7.接收方生成自己的随机私钥rRandPrv和随机数respNonce,然后将随机公钥rRandPub和respNonce封装成respAuthMsg返回给发起方.
	握手完成的最后,握手双方都会用各自得到的authMsg和respAuthMsg生成一组共享秘密(secrets),它其中包含了双方一致认同的AES密钥和MAC密钥等,这样此次信道上传输的信息都将用这组密钥来加密解密;
	共享秘密生成的过程类似于之前token产生的过程,双方都使用自己本地的随机私钥和对方的随机公钥相乘得到一个相同的密钥,再用这个密钥进行一系列Keccak256加密后得到AES密钥和MAC密钥.

newRLPXFrameRW

RLPx分帧,所有的读写都通过一个一个的帧(rlpxFrame)来传递和处理


	func newRLPXFrameRW(conn io.ReadWriter, s secrets) *rlpxFrameRW {
		macc, err := aes.NewCipher(s.MAC)
		if err != nil {
			panic("invalid MAC secret: " + err.Error())
		}
		encc, err := aes.NewCipher(s.AES)
		if err != nil {
			panic("invalid AES secret: " + err.Error())
		}
		// we use an all-zeroes IV for AES because the key used
		// for encryption is ephemeral.
		iv := make([]byte, encc.BlockSize())
		return &rlpxFrameRW{
			conn:       conn,
			enc:        cipher.NewCTR(encc, iv),
			dec:        cipher.NewCTR(encc, iv),
			macCipher:  macc,
			egressMAC:  s.EgressMAC,
			ingressMAC: s.IngressMAC,
		}
	}
	
2 doProtoHandshake

	func (t *rlpx) doProtoHandshake(our *protoHandshake) (their *protoHandshake, err error) {
		// 如果远程端有正当理由提前断开我们，我们应该将其作为错误返回，以便可以在其他地方进行跟踪。
		werr := make(chan error, 1)
		go func() { werr <- Send(t.rw, handshakeMsg, our) }()
		if their, err = readProtocolHandshake(t.rw, our); err != nil {
			<-werr // make sure the write terminates too
			return nil, err
		}
		if err := <-werr; err != nil {
			return nil, fmt.Errorf("write error: %v", err)
		}
		// 如果协议版本支持Snappy编码，请立即升级
		t.rw.snappy = their.Version >= snappyProtocolVersion
	
		return their, nil
	}

2.1 readProtocolHandshake

	func readProtocolHandshake(rw MsgReader, our *protoHandshake) (*protoHandshake, error) {
		msg, err := rw.ReadMsg()
		if err != nil {
			return nil, err
		}
		if msg.Size > baseProtocolMaxMsgSize {
			return nil, fmt.Errorf("message too big")
		}
		if msg.Code == discMsg {
			// 根据规范在协议握手有效之前断开连接，如果转发后检查失败，我们自己发送。
			// 
			var reason [1]DiscReason
			rlp.Decode(msg.Payload, &reason)
			return nil, reason[0]
		}
		if msg.Code != handshakeMsg {
			return nil, fmt.Errorf("expected handshake, got %x", msg.Code)
		}
		var hs protoHandshake
		if err := msg.Decode(&hs); err != nil {
			return nil, err
		}
		if len(hs.ID) != 64 || !bitutil.TestBytes(hs.ID) {
			return nil, DiscInvalidIdentity
		}
		return &hs, nil
	}