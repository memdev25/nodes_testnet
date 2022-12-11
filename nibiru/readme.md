Nibiru
==

Сайт: https://nibiru.fi/  
Дискорд: https://discord.com/invite/BVCw2cYmhu  
Github: https://gist.github.com/Jekins/2bf2d0638163f1294637#Headers

**1. Требования**  

    2 сpu
    4 GB RAM
    100 GB SSD


**2. Подготовка сервера**

    sudo apt update && sudo apt upgrade -y
    sudo apt install make clang pkg-config libssl-dev libclang-dev build-essential git curl ntp jq llvm tmux htop screen unzip cmake -y
    wget https://golang.org/dl/go1.19.2.linux-amd64.tar.gz
    sudo tar -C /usr/local -xzf go1.19.2.linux-amd64.tar.gz
    #Копируйте все вместе.
    cat <<EOF >> ~/.profile
    export GOROOT=/usr/local/go
    export GOPATH=$HOME/go
    export GO111MODULE=on
    export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
    EOF
    source ~/.profile
    go version
    rm -rf go1.19.2.linux-amd64.tar.gz

**3. Установка ноды**

    git clone https://github.com/NibiruChain/nibiru
    cd nibiru
    git checkout v0.15.0
    make install
    mv /root/go/bin/nibid /usr/bin/
    chmod +x /usr/bin/nibid
    nibid version
    #Должно показать версию v0.15.0
    
**4. Cоздаем кошелек**

    nibid init <moniker-name> --chain-id=nibiru-testnet-1 
    #Замените <moniker-name> на свое имя
    nibid keys add wallet
    #Команда запросит указать пароль, а затем выдаст мнемоник фразу, которую нужно будет обязательно сохранить в безопасное место.
    
Скачиваем genis.json фаил

    curl -s https://rpc.testnet-1.nibiru.fi/genesis | jq -r .result.genesis > genesis.json
    mv genesis.json $HOME/.nibid/config/genesis.json


 **5.Обновляем сonfig.toml фаил новыми пирами**

    cd ~
    nano .nibid/config/config.toml

Листаем вниз и ищем строчку persistent_peers и добавляем в нее пиры

    persistent_peers = "37713248f21c37a2f022fbbb7228f02862224190@35.243.130.198:26656,ff59bff2d8b8fb6114191af7063e92a9dd637bd9@35.185.114.96:26656,cb431d789fe4c3f94873b0769cb4fce5143daf97@35.227.113.63:26656"
    
**6. Скачиваем addressbook.**

    rm -rf .nibid/config/addrbook.json
    wget https://github.com/CryptoSailors/node-guides/releases/download/Nibiru/addrbook.json
    mv addrbook.json .nibid/config/
    
**7. Выставляем необходимые значения**

    sed -i 's/minimum-gas-prices =.*/minimum-gas-prices = "0.025unibi"/g' $HOME/.nibid/config/app.toml
    #Все что ниже копируем вместе и вставляем в терминал
    CONFIG_TOML="$HOME/.nibid/config/config.toml"
     sed -i 's/timeout_propose =.*/timeout_propose = "100ms"/g' $CONFIG_TOML
     sed -i 's/timeout_propose_delta =.*/timeout_propose_delta = "500ms"/g' $CONFIG_TOML
     sed -i 's/timeout_prevote =.*/timeout_prevote = "100ms"/g' $CONFIG_TOML
     sed -i 's/timeout_prevote_delta =.*/timeout_prevote_delta = "500ms"/g' $CONFIG_TOML
     sed -i 's/timeout_precommit =.*/timeout_precommit = "100ms"/g' $CONFIG_TOML
     sed -i 's/timeout_precommit_delta =.*/timeout_precommit_delta = "500ms"/g' $CONFIG_TOML
     sed -i 's/timeout_commit =.*/timeout_commit = "1s"/g' $CONFIG_TOML
     sed -i 's/skip_timeout_commit =.*/skip_timeout_commit = false/g' $CONFIG_TOML

**8. Cоздаем сервис фаил и запускаем ноду**

    #Копируем все вместе
    tee /etc/systemd/system/nibidd.service > /dev/null <<EOF
    [Unit]
    Description=Nibid
    After=network-online.target
    [Service]
    User=root
    ExecStart=/usr/bin/nibid start
    Restart=always
    RestartSec=3
    LimitNOFILE=10000
    [Install]
    WantedBy=multi-user.target
    EOF


**Настраиваем Pruning**

Открываем файл: root/.nibid/config/app.toml. 
Там корректируем значения pruning = "custom" вместо pruning = "default"

    pruning = "custom"
    # These are applied if and only if the pruning strategy is custom.
    pruning-keep-recent = "100"
    pruning-keep-every = "0"
    pruning-interval = "10"

Потом сохраняем файл и перезапускаем ноду  
Данное мероприятие начинает работать с текушего момента, ранее сохраненные блоки останутся на диске

Запускаем ноду

    systemctl daemon-reload
    systemctl start nibidd
    systemctl enable nibidd

Проверяем логи

    journalctl -u nibidd -f -n 100

**9. Запрашиваем токены из крана**

Пока нода синхронизируется, мы можем запросить токены из крана. Для этого нам понадобиться наш новый сгенерированный кошелек.

    nibid keys list

Идем в Discord и запрашиваем токены через Faucet

<img src="https://user-images.githubusercontent.com/18370861/206866395-8ec83b5e-211f-4683-826e-f7f1512f0167.png" width="500">

**10. Создаем Валидатора**

Для начал ждем полной синхронизации. Для того, что бы убедиться о том, что ваша нода засинкана, вбетей команду ниже.

Если команда выдаст true — значит синхронизация еще в процессе  
Если команда выдаст false — значит вы засинхронизированы и можете приступать к созданию валидатора.

    nibid status 2>&1 | jq .SyncInfo
    
![image](https://user-images.githubusercontent.com/18370861/206866290-6abd7ba2-03a2-4a6b-99fe-e68319556b07.png)

Cоздаем валидатора

    nibid tx staking create-validator --amount 1000000unibi --commission-max-change-rate "0.1" --commission-max-rate "0.20" --commission-rate "0.1" --min-self-delegation "1" --pubkey=$(nibid tendermint show-validator) --moniker <ВАШЕ_ИМЯ_ВАЛИДАТОРА> --chain-id nibiru-testnet-1 --gas-prices 0.025unibi --from wallet
    
Проверяем себя через эксплорер.

Достаем свой валлопер адресс и делегируем оставшиеся токены в своего валидатора.

    nibid keys show wallet --bech val
    nibid tx staking delegate <VAL_ADDRESS> 8000000unibi --chain-id nibiru-testnet-1 --from wallet --gas-prices 0.025unibi

**11. Сохрани свою ноду**

После успешного создания валидатора, вы должны позаботится забэкапить priv_validator_key.json. Без него вы не сможете востановить валидатора. Он находится в папке .nibid/config

![image](https://user-images.githubusercontent.com/18370861/206866354-bce2765b-4ae3-45fd-852d-3fdcfb394155.png)

**12. Удаление ноды**

    systemctl stop nibidd
    rm -rf /etc/systemd/system/nibidd.service
    rm -rf /usr/bin/nibid
    rm -rf nibiru
    rm -rf .nibid
    
 
Разное
-----------------------

Команды

https://nodejumper.io/nibiru-testnet/cheat-sheet

Снапшот

https://nodejumper.io/nibiru-testnet/sync


