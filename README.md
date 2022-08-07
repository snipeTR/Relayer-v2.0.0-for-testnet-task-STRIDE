# Mevcut kılavuzda, iki kozmos zinciri arasında IBC rölesinin nasıl kurulacağını öğreneceğiz.

Relayer-v2.0.0 yükleme ve çalıştırma örneğini kullanma

# Update system
```
     sudo apt update && sudo apt upgrade -y
```

# Bağımlılıkları yükle
```
     sudo apt install wget git make htop unzip -y
```
# Go 1.18.3 kurulumu
Gerekli ise kurun. zaten kurulu ise bu adımı atlayın.
```
     cd $HOME && \
     ver="1.18.3" && \
     wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
     sudo rm -rf /usr/local/go && \
     sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
     rm "go$ver.linux-amd64.tar.gz" && \
     echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
     source $HOME/.bash_profile && \
     go version
```
# Relayer in klasörünü oluşturma
```
     cd $HOME
     mkdir -p $HOME/.relayer/config
```

# go-v2 relayer indirme
```
     git clone https://github.com/cosmos/relayer.git
     cd relayer && git checkout v2.0.0
     make install
 
```
# Kendi ayarlarınıza göre değişken ayarlama.
aşağıdaki değerleri kendinize göre ayarlayarak yazın. portlarınızı örenmek için ilgili node'un config dosyalarına bakın.
```
     cd $HOME
     mkdir -p $HOME/.relayer/config
     
     MEMO=ornek_discord#1234
     
     KEYSTRIDE=stride için wallet ismi
     KEYGAIA=gaia için wallet ismi
     
     IPSTRIDE=stride sunucu ip adresi #123.123.123.123
     PORTSTRIDE=STRIDE icin RPC portu nodunuza göre ayarlayın.

     IPGAIA=IP you GAIA node #123.123.123.123
     PORTGAIA=GAIA icin RPC portu nodunuza göre ayarlayın.
```

# Next - zincir yapılandırma dosyalarını manuel olarak eklemek için
bir sonraki adımı kullanacaksanız bu adımı atlayın.
```
$ rly chains add --url https://gist.githubusercontent.com/Archebald-now/3aef116b9dd67009600d8da1746dfe1f/raw/06f7e8959d5d9735576867ae723ca5c35f485aed/GAIA_config.json gaia
$ rly chains add --url https://gist.githubusercontent.com/Archebald-now/3aef116b9dd67009600d8da1746dfe1f/raw/06f7e8959d5d9735576867ae723ca5c35f485aed/STRIDE-TESTNET-2_config.json stride
```
bundan sonra yapılandırma dosyasında değişiklik yapmanız gerekir
```nano /root/.relayer/config/config.yaml```
ve bilginizle rpc-addr'yi değiştirin

# Veya - tüm komutu terminale kopyalayın
not: aşağıdaki tüm satırlar tek bir komuttur. ve tek bir seferde yapıştırılmalıdır.
```

sudo tee $HOME/.relayer/config/config.yaml > /dev/null <<EOF
global:
    api-listen-addr: :5183
    timeout: 10s
    memo: $MEMO
    light-cache-size: 20
chains:
    GAIA:
        type: cosmos
        value:
            key: $KEYGAIA
            chain-id: GAIA
            rpc-addr: http://$IPGAIA:$PORTGAIA
            account-prefix: cosmos
            keyring-backend: test
            gas-adjustment: 1.3
            gas-prices: 0.01uatom
            debug: false
            timeout: 10s
            output-format: json
            sign-mode: sync
            strategy:
            type: native
            version: ics20-1
            order: UNORDERED
    stride:
        type: cosmos
        value:
            key: $KEYSTRIDE
            chain-id: STRIDE-TESTNET-2
            rpc-addr: http://$IPSTRIDE:$PORTSTRIDE
            account-prefix: stride
            keyring-backend: test
            gas-adjustment: 1.3
            gas-prices: 0.01ustrd
            debug: false
            timeout: 20s
            output-format: json
            sign-mode: sync
            strategy:
            type: native
            version: ics20-1
            order: UNORDERED
paths:
    stride-gaia:
        src:
            chain-id: STRIDE-TESTNET-2
            client-id: 07-tendermint-0
            connection-id: connection-0
            port-id: icacontroller-GAIA.DELEGATION,transfer,icacontriiler-GAIA.FEE,icacontroller-GAIA.WITHDRAWAL,icacontroller-GAIA.REDEMPTION
  
        dst:
            chain-id: GAIA
            client-id: 07-tendermint-0
            connection-id: connection-0
            port-id: icahost,icacontroller-GAIA.DELEGATION,transfer,icacontriiler-GAIA.FEE,icacontroller-GAIA.WITHDRAWAL,icacontroller-GAIA.REDEMPTION
        src-channel-filter:
            rule: allowlist
            channel-list:
                - channel-0 
                - channel-1 
                - channel-2 
                - channel-3 
                - channel-4 
        dst-channel-filter:
            rule: allowlist
            channel-list:
                - channel-0 
                - channel-1 
                - channel-2 
                - channel-3 
                - channel-4
EOF
```

# Röleye eklenen zincirleri kontrol edin
   ```
   rly chains list
   ```
# Başarılı çıktı:
```
 1: STRIDE-TESTNET-2     -> type(cosmos) key(✔) bal(✔) path(✔)
 2: GAIA                 -> type(cosmos) key(✔) bal(✔) path(✔)
```
# "Path" un doğru olup olmadığını kontrol edin
   ```
   rly paths list
   ```
# Başarılı çıktı:
```
0: gaia-stride          -> chns(✔) clnts(✔) conn(✔) (GAIA<>STRIDE-TESTNET-2)
```

# Aktarıcının işlemleri imzalarken ve aktarırken kullanması için anahtarları içe aktarın
   ```
     rly keys restore stride $KEYSTRIDE "mnemonic words here"
     rly keys restore gaia $KEYGAIA "mnemonic words here"
   ```
# Cüzdan bakiyesini kontrol edin
stride hesabınızda STRD gaia hesabınızda ATOM paranızdan bir miktar olmalı.
```
rly q balance stride
rly q balance gaia
```
   
# go-v2 aktarıcı hizmet dosyası oluşturun
 (tek bir komutla terminale kopyalayıp yapıştırın)
```
sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF
[Unit]
Description=Relayer_v2
After=network.target
[Service]
Type=simple
User=$USER
ExecStart=$(which rly) start gaia-stride --log-format logfmt --processor events
Restart=on-failure
RestartSec=3
LimitNOFILE=4096
[Install]
WantedBy=multi-user.target
EOF
```

# Hizmeti başlat
```
     sudo systemctl daemon-reload
     sudo systemctl enable rlyd
     sudo systemctl restart rlyd && journalctl -fu rlyd -o cat
```
# Aşağıdaki günlükler, Relayer_v2.0.0 kurulumunun başarıyla tamamlandığını gösterecektir:
<a href='https://postimg.cc/XBGzBqFv' target='_blank'><img src='https://i.postimg.cc/XBGzBqFv/logs-relayer-v2.jpg' border='0' alt='logs-relayer-v2'/></a>

# Bu kılavuz için ilham kaynağı olan goooodnes#8929 ve Zuka#5870'e teşekkürler.
