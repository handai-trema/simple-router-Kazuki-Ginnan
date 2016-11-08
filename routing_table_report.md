#第5回課題:ルータのCLIを作ろう

氏名:銀杏一輝  
学籍番号:33E16006  
提出日:2016/11/08  

書き換えたコード一覧  
* [simple_router.rb](https://github.com/handai-trema/simple-router-Kazuki-Ginnan/blob/develop/lib/simple_router.rb)
* [routing_table.rb](https://github.com/handai-trema/simple-router-Kazuki-Ginnan/blob/develop/lib/routing_table.rb)
* [interface.rb](https://github.com/handai-trema/simple-router-Kazuki-Ginnan/blob/develop/lib/interface.rb)
* [simple_router](https://github.com/handai-trema/simple-router-Kazuki-Ginnan/blob/develop/bin/simple_router)(コマンドの記述)

#ルーティングテーブルの表示
simple_router.rbに以下のプログラムを記述した。

```
  def show_table()
      show_routing_tables() 
  end

       (省略)
  def show_routing_tables()
    @routing_table.show_tables()
  end
```
routing_table.rbに以下のプログラムを記述した。

```
def show_tables()
       puts "Destinaton_address/netmask   next_hop\n"
    MAX_NETMASK_LENGTH.downto(0).each do |each|
      @db[each].each do |key, value|
       puts "#{IPv4Address.new(key).to_s}/#{each.to_s} \t #{value.to_s} \n"
      end
    end
  end
```

まず、ネットマスク長ごとに探索し、各ネットマスク長で、キー(key)と値(value)ごとに探索している。keyはstring型で表示するため、IPv4Addressのクラスを用いて新たにセットしている。keyはdestination_addressになっており、valueがnext_hopになっている。(2重のループの部分が中々思い浮かばなかったため、[辻健太](https://github.com/handai-trema/simple-router-k-tsuji/blob/develop/report_simple-router.md)さんのレポートを参考させていただきました。ありがとうございます。)


simple_routerに以下のコマンドを追加した。

```
desc 'Show a routing table'
  arg_name ''
  command :show_table do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
    c.action do |_global_options, options, args|
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
        show_table()
    end
  end
```

実行結果は以下のようになった。trema.conf/simple_router.confは配布時のものをそのまま利用している。

```
./bin/simple_router show_table

Destinaton_address/netmask   next_hop  (別のターミナルの表示)
0.0.0.0/0 	 192.168.1.2 
```


#ルーティングテーブルエントリの追加と削除

ルーティングテーブルの追加に関して以下のプログラムを追加した。  

simple_router.rbに以下のコードを追加した。
```
def add_entry(dst_addr, net_mask, next_hop)
      hash = {:destination => dst_addr, :netmask_length => net_mask, :next_hop => next_hop}
      @routing_table.add(hash)
  end
```
hashはハッシュを生成するリテラルを表しており、hash ={A => B,・・・}でAはキー、Bが値を表している。
([参考サイト1](https://docs.ruby-lang.org/ja/latest/class/Hash.html))  
@routing_table.add(hash)はもともと記述されたものを使っており、これにより、ルーティングテーブルに追加する。  


simple_routerに以下のコマンドを追加した。
```
desc 'Add a entry'
  arg_name 'dst_addr net_mask next_hop'
  command :add_entry do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      dst_addr = args[0].to_s
      net_mask = args[1].to_i
      next_hop = args[2].to_s
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
        add_entry(dst_addr, net_mask, next_hop)
    end
  end
```

実行結果は以下のようになった。trema.conf/simple_router.confは配布時のものをそのまま利用している。

```
$ ./bin/simple_router show_table 

Destinaton_address/netmask   next_hop     (別のターミナルの表示)
0.0.0.0/0 	 192.168.1.2 

$ ./bin/simple_router add_entry 192.168.1.1 24 192.168.2.1
$ ./bin/simple_router add_entry 192.168.2.1 24 192.168.2.1
$ ./bin/simple_router add_entry 192.168.3.1 24 192.168.2.1
$ ./bin/simple_router show_table 

Destinaton_address/netmask   next_hop     (別のターミナルの表示)
192.168.1.0/24 	 192.168.2.1 
192.168.2.0/24 	 192.168.2.1 
192.168.3.0/24 	 192.168.2.1 
0.0.0.0/0 	 192.168.1.2 
```



ルーティングテーブルの削除は以下のようなプログラムを追加した。  

routing_table.rbに以下のテーブルを削除するコードを追加した。

```
def delete(options)
    netmask_length = options.fetch(:netmask_length)
    prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
    @db[netmask_length].delete(prefix.to_i)
  end
```

add(option)にならって作成した。  
ハッシュの削除に関しては、deleteのメソッドが用いることにより、実装した([参考サイト2](http://www.rubylife.jp/ini/hash_class/index5.html))。(インスタンス変数でのdeleteの使い方が中々思い浮かばなかったため、[辻健太](https://github.com/handai-trema/simple-router-k-tsuji/blob/develop/report_simple-router.md)さんのレポートを参考にさせていただきました。ありがとうございます。)  


simple_router.rbに以下のコードを追加した。
```
  def delete_entry(dst_addr, net_mask, next_hop)
      hash = {:destination => dst_addr, :netmask_length => net_mask, :next_hop => next_hop}
      @routing_table.delete(hash)
  end
```

simple_routerに以下のコードを追加した。
```
 desc 'Delete a entry'
  arg_name 'dst_addr net_mask next_hop'
  command :delete_entry do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      dst_addr = args[0].to_s
      net_mask = args[1].to_i
      next_hop = args[2].to_s
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
        delete_entry(dst_addr, net_mask, next_hop)
    end
  end
```

実行結果は以下のようになった。trema.conf/simple_router.confは配布時のものをそのまま利用している。  
上記のルーティングテーブルの追加の実行の後にコマンドを実行した。

```
$ ./bin/simple_router show_table

Destinaton_address/netmask   next_hop (別のターミナル)
192.168.1.0/24 	 192.168.2.1 
192.168.2.0/24 	 192.168.2.1 
192.168.3.0/24 	 192.168.2.1 
0.0.0.0/0 	 192.168.1.2
 
$ ./bin/simple_router delete_entry 192.168.1.1 24 192.168.2.1
$ ./bin/simpouter show_table 

Destinaton_address/netmask   next_hop　　(別のターミナル)
192.168.2.0/24 	 192.168.2.1 
192.168.3.0/24 	 192.168.2.1 
0.0.0.0/0 	 192.168.1.2 

$ ./bin/simple_router delete_entry 192.168.2.1 24 192.168.2.1
$ ./bin/simple_router show_table 

Destinaton_address/netmask   next_hop　　(別のターミナル)
192.168.3.0/24 	 192.168.2.1 
0.0.0.0/0 	 192.168.1.2 

$ ./bin/simple_router delete_entry 192.168.3.1 24 192.168.2.1
$ ./bin/simple_router show_table 

Destinaton_address/netmask   next_hop　　　(別のターミナル)
0.0.0.0/0 	 192.168.1.2 
```
#ルータのインタフェース一覧の表示

interfaces.rbに以下のコードを追加した。

```
 def self.show_interfaces()
     puts "INTERFACES =[\n"
     all.find do |each|
      puts "{\n"
      puts "port_num = #{each.port_number}"
      puts "mac_address = #{each.mac_address}"
      puts "ip_address = #{each.ip_address.to_s}"
      puts "netmask_length = #{each.netmask_length}"
      puts "}\n"
     end
     puts "]\n"
  end
```

ポート番号、MACアドレス、IPアドレス、ネットマスクはattr_readerにより、port_number,mac_address,ip_address,netmask_lengthにより参照できるので、それを用いて出力できるようにした。

simple_router.rbに以下のコードを追加した。
```
  def show_interfaces()
      Interface.show_interfaces()
  end
```

simple_routerに以下のコマンドを追加した。
```
  desc 'Show router interfaces'
  arg_name ''
  command :show_interfaces do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
        show_interfaces()
    end
  end
```

実行結果は以下のようになった。trema.conf/simple_router.confは配布時のものをそのまま利用している。
```
$ ./bin/simple_router show_interfaces

INTERFACES =[              (別のターミナル)
{
port_num = 1
mac_address = 01:01:01:01:01:01
ip_address = 192.168.1.1
netmask_length = 24
}
{
port_num = 2
mac_address = 02:02:02:02:02:02
ip_address = 192.168.2.1
netmask_length = 24
}
]


```




