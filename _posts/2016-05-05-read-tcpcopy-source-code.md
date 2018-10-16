---
layout: post
date: 2016-05-05T10:00:25+08:00
title: Tcpcopy 源码阅读
tags: 源码阅读
---

[tcpcopy](https://github.com/session-replay-tools/tcpcopy)是网易开源的一款压测工具，可以实时复制线上流量到测试环境，从而利用线上真实的用户流量来对测试环境进行仿真压测。
由于网上关于tcpcopy的介绍并不多，我对tcpcopy的原理也很感兴趣。因此在学习其源码后写下这篇文章和大家分享。这里非常感谢tcpcopy的作者王斌老师的热心指导。


# 架构
第一种架构：

* 实现原理：
  * 从数据链路层捕获客户端的请求包，修改目的IP地址为压测机器，再从IP层发送出去。
  * 从数据链路层捕获压测机器的响应包，保持tcp会话的状态（seq number，ack number等），以此来欺骗压测机器的TCP协议栈。

* 图解： 
  <img src="/assets/images/read-tcpcopy-source-code/illustration-1.png" width="800" />
  <img src="/assets/images/read-tcpcopy-source-code/illustration-2.png" width="800" />

* 局限：
  * 只支持同一网段：tcpcopy通过网卡的混杂模式来抓包，因此除非在路由设备上设置强制路由，否则响应包无法回到tcpcopy的机器。
  * 难以支持多台现网机器流量复制：把在线请求包导到压测机器进行压测的关键在于欺骗压测机器的TCP协议栈，因此我们需要在tcpcopy的进程捕获压测机器的响应包，保存相关字段来维持和压测机器之间会话的状态。如果有多个tcpcopy机器，那压测机器需要维护多个tcpcopy机器的请求包关系，才能把响应包回给响相应的tcpcopy机器，只有当tcpcopy机器能正常收到压测机器的响应包，才能维护正确的TCP会话状态。而目前的架构压测机器无法得知请求包来自哪个tcpcopy机器，自然也没办法将响应包回给正确的tcpcopy机器。
  为什么不把请求包的源IP替换成tcpcopy机器的IP呢？这样压测机器就能把响应包准确回复给相应的tcpcopy的机器。但如果这样的话，由于tcpcopy机器并没有监听该端口，会发送reset包给压测机器，连接将被断开，压测也无法进行下去。
  

# 流程

截获客户端请求包的处理：

1.  如果是SYN/RST包，转发一份到压测机器。
2.  如果是FIN包，判断下之前的数据包压测机器是否已经确认，如果都确认了，就转发FIN包给压测机器，如果还有没确认的，则先把FIN包保存下来。
3.  如果是普通数据包，如果当前是SYN_SENT状态，则先保存到unsend列表里，否则转发到压测机器。

截获压测机器响应包的处理：

1.  如果是reset包，设置reset_flag为true。
2.  如果不是ack包，不需要处理。
3.  保存响应包的ack_seq到virtual_ack字段。
4.  如果是SYN包，更新virtual_next_sequence字段的值为响应包的seq number加1，更新virtual_status为SYN_CONFIRM，三次握手完成，将unsend的数据包（如果有）转发给压测机器。
5.  如果是FIN包，更新virtual_status为SERVER FIN，更新virtual_next_sequence字段的值为响应包的seq number加1。如果有fin_ack_packge，则将该fin_ack_package发送给压测机器。
6.  如果是数据包，根据头部的信息更新virtual_next_sequence。

# 源码细节
    
```C++
struct session_st {
    uint32_t virtual_next_sequence; //下个要发送的segment的ack sequence number
    uint32_t virtual_ack; //收到segment的ack sequence number
    uint16_t virtual_status; //当前状态：Client Fin, Server Fin, SYN_CONFIRM, SYN_SENT
    uint32_t client_next_sequence; //下个要发送的segment的sequence number
    uint16_t client_window; // 窗口大小
    uint16_t client_ip_id ;  //包的序列号
    unsigned char *fin_ack_package; //fin ack包
    bool     reset_flag; //是否重置
    dataContainer unsend; //segment缓存队列
｝


int main(int argc ,char **argv)
{
    ...
    //创建收发二层（链路层）报文的raw socket，链路层抓包
    int sock = socket(AF_PACKET,SOCK_RAW,htons(ETH_P_ALL));
    
    //创建收发三层（IP层）报文的raw socket，IP层转发包
    send_init();
    while(1)
    {
        //在链路层捕获到客户端请求包或压测机器的回包时，由process函数处理
        int recv_len = recvfrom(sock,recvbuf,2000,0,NULL,NULL);
        process(recvbuf,recv_len);
    }
    return 0;
}

void process(char *packet,int len)
{
    struct etharp_frame *ether = (struct etharp_frame *)packet;
    ...
    //判断以太网报文的类型是不是IP
    if(ntohs(ether->type) != 0x800){
        return;
    }
    ip_header = (struct iphdr*)(packet+sizeof(struct etharp_frame ) );
    //判断是不是TCP
    if(ip_header->protocol != IPPROTO_TCP)
    {
        return ;
    }
    
    size_ip = ip_header->ihl*4; //获取ip头部的大小
    tcp_header = (struct tcphdr*)((char *)ip_header+size_ip); 
    size_tcp = tcp_header->doff*4; //获取tcp头部的大小

        
    if( (ip_header->saddr==remote_ip) && (tcp_header->source==remote_port) )
    {
        //如果是压测机器的回包，由update_virtual_status函数处理
        seIterator iter = sessions.find(get_ip_port_value(ip_header->daddr,tcp_header->dest));
        if(iter != sessions.end())
        {
            iter->second.update_virtual_status(ip_header,tcp_header);
            if( iter->second.is_over())
            {
                sessions.erase(iter);
            }
        }
    }
    else if( (ip_header->daddr==local_ip) && (tcp_header->dest==local_port))
    {
        //如果是客户端请求包，由process_recv函数处理
        if(tcp_header->syn)
        {
            sessions[get_ip_port_value(ip_header->saddr,tcp_header->source)].process_recv(ip_header,tcp_header);
        }
        else
        {
            seIterator iter = sessions.find(get_ip_port_value(ip_header->saddr,tcp_header->source));
            if(iter != sessions.end())
            {
                iter->second.process_recv(ip_header,tcp_header);
                if ((iter->second.is_over()))
                {
                    sessions.erase(iter);
                }
            }
        }
    }
}

void session_st::update_virtual_status(struct iphdr *ip_header,struct tcphdr* tcp_header)
{
    if( !(tcp_header->ack))
    {
        return; //不是ack不处理
    }
    //记录压测机器回包的ack seq number
    virtual_ack = tcp_header->ack_seq;
    //处理syn包
    if( tcp_header->syn)
    {
        //virtual_next_sequence加1
        virtual_next_sequence = plus_1(tcp_header->seq);
        virtual_status |= SYN_CONFIRM;
        //如果缓存队列有未发送的包，则转发到压测机器
        while(! unsend.empty())
        {
            unsigned char *data = unsend.front();
            send_ip_package(data, virtual_next_sequence);
            free(data);
            unsend.pop_front();
        }
        return;
    }
    //处理fin包
    else if(tcp_header->fin)
    {
        virtual_status  |= SERVER_FIN;
        //virtual_next_sequence加1
        virtual_next_sequence = plus_1(tcp_header->seq);
        //如果有保存fin_ack_package，则发送出去。
        if(fin_ack_package)
        {
            send_ip_package(fin_ack_package,virtual_next_sequence);
        }
        return;
    }
    uint32_t tot_len = ntohs(ip_header->tot_len);
    uint32_t next_seq = htonl(ntohl(tcp_header->seq)+tot_len-ip_header->ihl*4-tcp_header->doff*4);
    
    if(ntohl(next_seq) < ntohl(virtual_next_sequence))
    {
        //如果next_seq小于virtual_next_sequence，意味着之前给压测机器的ack包丢了，重新发送ack包
        send_fake_ack(ip_header->daddr,tcp_header->dest);
    }
    else if(ntohl(next_seq)==ntohl(virtual_next_sequence))
    {
        //has data
        if(tot_len != ip_header->ihl*4+tcp_header->doff*4)
        {
            //同上，ack包丢了，重发
            send_fake_ack(ip_header->daddr,tcp_header->dest);
        }
    }
    else
    {
        virtual_next_sequence = next_seq;
    }

    /* 如果收到客户端的fin包，但压测机器还有回包没有ack，就会把客户端的fin包保存在fin_ack_package。当virtual_ack等于client_next_sequence，意味着服务器已经确认了所有数据，如果fin_ack_package不为空，意味着客户端没有数据要发送了，因此将之前保存的fin包发送给压测机器。
    */
    if(virtual_ack == client_next_sequence)
    {
        if(fin_ack_package)
        {
            send_ip_package(fin_ack_package,virtual_next_sequence);
            virtual_status |= CLIENT_FIN;
            confirmed = true;
        }
    }
}

void session_st::process_recv(struct iphdr *ip_header,struct tcphdr *tcp_header)
{
    //syn包
    if(tcp_header->syn)
    {
        send_ip_package((unsigned char *)ip_header,virtual_next_sequence);
        return;
    }
    //fin包
    if(tcp_header->fin)
    {
        //virtual_ack等于tcp_header->seq表示压测机器已经ack了所有请求数据
        if(virtual_ack == tcp_header->seq)
        {
            send_ip_package((unsigned char *)ip_header,virtual_next_sequence);
            virtual_status |= CLIENT_FIN;
            return;
        }
        else
        { 
            //如果压测机器还有未确认的请求包，则先保存客户端的fin包
            fin_ack_package = copy_ip_package(ip_header);
        }
        return;
    }
    //更新client_next_sequence	
    save_header_info(ip_header,tcp_header);
    //如果三次握手还没完成，先将数据包压入缓存队列
    if(virtual_status == SYN_SEND)
    {
        unsend.push_back(copy_ip_package(ip_header));
    }
    else
    {
        //否则直接转发给压测机器
        send_ip_package((unsigned char *)ip_header,virtual_next_sequence); 
    }
}
```

