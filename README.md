1. 環境一個對外的IP<CHE_HOST = xxx.network.com>
2. 連線網路透過一台路由器
3. che server內網ip = 192.168.0.x，需將路由把8080及5050 port 開給che server。
    8080是che Server的預設port，如需更改請在run docker時加上 -e CHE_PORT=10xxx
    5050是keycloak的預設port，透過keycloak的登入模組連到cheServer，從外部輸入網址xxx.network.com:8080其實會轉到xxx.network.com:5050的登入畫面。
    因為會透過session連線，需產生必要的金鑰就需透過keycloak取得。在預設安裝下如果只是要給內網使用，有發現到che要求CHE_KEYCLOAK_AUTH__SERVER__URL
    路徑有點問題。如果是內網使用只需要容器間的橋接有連到就可以了，但是他預設的位址跟che預設安裝結果的IP有點出入，預設是http://192.168.0.x:5050/auth。
    變成是跟docker0做連結了，但che容器的netWork一開始也沒把bridge加進來變成要麻手動加進來就或是在run時加上 -e CHE_KEYCLOAK_AUTH__SERVER__URL=
    http://172.18.0.3/auth 直接在容器內做連結就行了。
    註:ip可能會因設定或其他情況有所不同，在輸入密碼後卡在畫面時就到che的logs看一下是去那要金鑰了。
4.  che server(192.168.0.x)設定iptables 要對外開放8080 及 5050 的port。註$EIF為你的網卡名稱
iptables -A INPUT -i $EIF -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -i $EIF -p tcp --dport 5050 -j ACCEPT
    跟localhost一樣要把docker0的連線都打開
iptables -A INPUT -i docker0 -j ACCEPT
iptables -A OUTPUT -o docker0 -j ACCEPT
    multi mode 在起workspace時會建立eclipse/ubuntu_jdk8:latest的container，會包含有container要用的tomcat、Terminal...等，看在add WorkSpace時選擇那幾種stack。預設是從32768 ~ 65535的備用port去開，所以有設定路由的話要這範圍的port將導回che server。
5. 執行docker run
docker run -it --rm -e CHE_MULTIUSER=true \
                    -e CHE_HOST=dogtoo.mynetgear.com \
                    -e CHE_KEYCLOAK_AUTH__SERVER__URL=http://xxx.network.com:5050/auth \
                    -v /var/run/docker.sock:/var/run/docker.sock 
                    -v ~/eclipse:/data eclipse/che:6.1.1 start
    CHE_MULTIUSER : 設定為多人使用版本
    CHE_HOST : 對外連線IP
    CHE_KEYCLOAK_AUTH__SERVER__URL : 因為是對外連線，所以要告訴連進來的使用者要去那要金鑰
    ~/eclipse : 本地的data資料，也就是docker的VOLUME
    che:6.1.1 : che的版本，看自已要用那一版
6. run時可能會遇到的問題
conn (browser => ws):    [NOT OK]
conn (server => ws):     [NOT OK]
    到~/eclipse的資料夾下可以看到cli.log可以看到
/usr/bin/curl  "-I localhost:32768/alpine-release -s -o /dev/null --connect-timeout 5 --write-out %{http_code}"28
/usr/bin/curl  "-I xxx.network.com:32768/alpine-release -s -o /dev/null --connect-timeout 5 --write-out %{http_code}"28
         conn (browser => ws):    [NOT OK]
         conn (server => ws):     [NOT OK]    
    在啟動時去對32768做了檢測，這時就先把對外的32768打開
iptables -A INPUT -i $EIF -p tcp --dport 32768 -j ACCEPT
    再重新跑應該就不會有問題了。
7. keycloak的Valid Redirect URIs，Server啟動後預設會是內網的ip，也有可能你第一次設定就外網就不會有問題。所以連到
    http://dogtoo.mynetgear.com:5050/auth/admin登入admin，預設密碼為admin記得到Account改密碼。進入後點選「Clients」會看到「che-public」
    將「Valid Redirect URIs」及「Web Origins」改成
http://dogtoo.mynetgear.com:8080/*
http://dogtoo.mynetgear.com:8080
    即可。
