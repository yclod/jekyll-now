---
layout: post
title: How to connect NAT IP inside
---

NAT指的是[Network address translation](http://zh.wikipedia.org/wiki/%E7%BD%91%E7%BB%9C%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2)，NAT IP就是指外部访问一台机器用的IP地址，比如10.10.123.123，也是通过DNS解析域名后得到的IP地址，但是这台机器在网络内部，其实用的是另外一个IP，比如192.168.123.123, 这时有的程序会有问题，比如最近在使用的Splunk server的[Modular Input](https://github.com/damiendallimore/SplunkModularInputsJavaFramework/blob/master/src/com/splunk/modinput/ModularInput.java), 代码的第306行尝试使用服务器的名称连接socket:

<pre><code class="language-java">
public void run() {
			try {

				int failCount = 0;
				while (true) {
					Socket socket = null;
					connectedToSplunk = false;
					try {
						socket = new Socket(this.splunkHost, this.port);
						connectedToSplunk = true;
						failCount = 0;
					} catch (Exception e) {
						logger.error("Probing socket connection to SplunkD failed.Either SplunkD has exited ,or if not,  check that your DNS configuration is resolving your system's hostname ("
								+ this.splunkHost
								+ ") correctly : "
								+ e.getMessage());
						failCount++;
						connectedToSplunk = false;
					} finally {
						if (socket != null)
							try {
								socket.close();
							} catch (Exception e) {
							}
					}
					if (failCount >= 3) {
						logger.error("Determined that Splunk has probably exited, HARI KARI.");
						System.exit(2);
					} else {
						Thread.sleep(10000);
					}

				}

			} catch (Exception e) {
			}

		}
</code></pre>

这里this.splunkHost的返回值是形似xxx.yyy.com的域名，使用域名（xxx.yyy.com）来连接自身的socket，域名会被DNS解析成形如10.10.123.123的外部IP地址，而外部的IP连接自身是连接不上的，因为连接自己没有走外部路由，不知道10.10.123.123这个地址，所以socket连接会出现:Connect Timeout的错误。

解决这个问题，我们有一个简单的方法，就是尝试修改/etc/hosts,让其将域名和自身的内部IP关联上，具体做法是添加如下行到/etc/hosts中：
<pre><code>
127.0.0.1 xxx xxx.yyy.com
</code></pre>

