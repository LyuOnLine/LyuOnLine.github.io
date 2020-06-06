### NAT及其行为
  
不同NAT的行为的不同是NAT穿越的主要困难。[bcp127](https://www.rfc-editor.org/bcp/bcp127)给出了NAT行为的相关术语，及建议实现。

- 内部地址
  - NAT内部地址，由内部IP，端口组成。记为(in_ip: in_p)
  - NAT映射外部地址。记为(out_ip:out_p)
  
- NAT会话(session):
    在转发内部(in_ip:in_p)到外部(out_ip:out_p)的过程中，NAT会记录并维持此连接，这被称为NAT会话。 
- 地址与端口映射(address and port mapping)
   - 在内部(in_ip:in_p)已经与(IP1:PORT1)建立NAT会话情况下。相同内部地址(in_ip:in_p)与外一外部地址(IP2:PORT2)建立新的NAT对话时，NAT的不同行为。
     - 端点无关映射: 直接使用原来NAT会话。既，虽然外部地址不同，但(in_ip:in_p)到(out_ip,out_p)的对应关系保持不变。
     - 地址相关映射: 当外部地址不变时，才使用原来NAT会话。既，如果IP1 != IP2， 会生成新的会话: (in_ip:in_p) -> (out_ip2:out_p2)
     - 地址端口相关映射: 只有当外部地址端口均不变时，才使用原来NAT会话。
   - **RFC建议**: 
     - 必须实现端口无关映射。
 - 地址池(ip pool)使用
   - 如果NAT有多个IP地址可用时。同一内部IP不同端口对应的外部IP分配策略。
     - 任意分配(arbitrary)： 同一内部IP，可能会对应多个不同的外部IP。
     - 匹配分配(paired): 同一内部IP，映射的外部IP保持不变。
   - **RFC建议**: 
     - 如果NAT拥有地址池，应使用匹配分配策略。
 - 端口分配：
   - 外部端口(out_p)与内部端口(in_p)的对应关系，NAT行为可分成以下几类:
     - 端口保留(port perservation):
        ```python
        if out_p空闲:
          out_p = in_p
        else:
          out_p != in_p
        ```
      - 端口超载(port overloading)
        - 确保out_p = in_p，不管out_p是否被占用。
        - 如果out_p已被分配，旧session状态会丢弃。
      - no port perservation:
        - 不会尝试端口一致性
  
 - 端口奇偶性:
   - 外部端口(out_port)是否保持与内部端口(in_port)的奇偶性相同
   - **RFC建议**:
     - 建议保持端口的奇偶性。
 - 端口连续性:
   - 如果in_p是连续的，out_p是否保持连续性。
   - rfc不要求。
 - 映射刷新(mapping refresh)
   - 指NAT映射的timeout逻辑。在NAT没有数据通信时，会在多长时间保持此映射的有效性。
   - **RFC建议**:
     - UDP映射应至少保持在2分钟，不会失效。(0-1023端口，可以按INNA标准的延时逻辑，可以不受此限制)
     - UDP映射的timeout值，可以是用户可修改的。
     - UDP映射timeout，建议默认值为5分钟或更大。
 - 映射刷新方向
   - NAT outbound refresh: 在NAT映射建立后，向外发送的包，是否会刷新mapping timeout计时器。
   - NAT inbound refresh: 在NAT映射建立后，由外向内网传递的包，是否会刷新mapping timeout计时器。
   - **RFC建议**:
     - NAT必须outbound refresh为True。以允许client自己刷新保持连接。
     - NAT可以inbound refresh为True。
 - 过滤行为
   - 在内网计算机通过(in_ip:in_p):(out_ip:out_p)映射，与外部IP(X:x)建立连接后；另一个外部IP(Y:y)是否允许使用此映射
   - 端点无关过滤(endpoint independent filtering)
     - (Y:y)允许使用为(X:x)建立的映射，建立与内网的通信。
   - 地址相关过滤(address dependent filtering)
     - 如果地址相同 (Y=X)，则允许通信。
   - 地址、端口相关过滤(address and port dependent filtering)
     - 只有当 (Y=X and y=x)时，才允许通信。
   - **RFC建议**:
     - 如果透明最重要，建议使用端口无关过滤。以使从多个地址接收media，及使用集合成为可能。
     - 如果采用更严格的过滤，应使用地址相关过滤。
 - 回形针形为(hairpinning)
   - 假设NAT为内网IP(in_ip:ip_p)与(out_ip:out_p)建立了映射。内部IP端口(in_ip:ip_p2)是否能够通过(out_ip:out_p)与自己通讯。
   - **RFC建议**:
     - 应支持回形针通讯。
- 参考:
  - [bcp127](https://www.rfc-editor.org/bcp/bcp127)
 
### STUN服务器
- STUN服务器功能:
  - 获取外网IP(out_ip:out_port)
  - 获取NAT行为:  
    - 映射行为 : {端口无关映射， 地址相关映射， 地址端口相关映射}
    - 过滤行为 : {端点无关过滤, 地址相关过滤, 地址、端口相关过滤}
    - 端口分配 : {端口保留, 端口超载, no port perservation}
    - 是否支持回形针。
- 检测方法:
  - 映射行为判断:
    - 向stun server的两个IP（甚至向两个stun server)发送bind请求，判断映射地址是否相同。
    - 此判断，似乎可以省略(?)
      - [stund](https://github.com/hanpfei/stund)源码中，此步实际使用相同IP。或者说，没有判断。
      - google的stun server的两个IP和端口并不连续。端口分别为19302, 19305。          
      - [stun_client.py](https://github.com/LyuOnLine/test_examples/tree/master/webrtc/stun)，使用两个server进行判断。
      - rfc要求NAT必须是端口无关映射。
  - 端口过滤行为:
    - 向stun server发送以下3种bind请求:
      ```
      I:   bind(change_ip=0, change_port=0)
      II:  bind(change_ip=1, change_port=0)
      III: bind(change_ip=0, change_port=1)
      ```  
      - 如果I,II收到： 端口无关。
      - 如果I,III收到, II收不到：  地址相关过滤。
      - 如果I收到， II,III收不到：  地址端口相关过滤。

### STUN协议实例

![stun_protocol](resource/stun_client.svg)


- 参考
  - [STUN (RFC 3489) vs. STUN (RFC 5389/5780)](https://www.netmanias.com/en/post/techdocs/6065/nat-network-protocol/stun-rfc-3489-vs-stun-rfc-5389-5780)
	- [NAT Behavior Discovery Using STUN (RFC 5780)](https://www.netmanias.com/en/post/techdocs/6067/nat-network-protocol/nat-behavior-discovery-using-stun-rfc-5780)
	- [rfc3489](https://tools.ietf.org/html/rfc3489)
	- [mine stun clinet](https://github.com/LyuOnLine/test_examples/tree/master/webrtc/stun)
	- [stund](https://github.com/hanpfei/stund)
