```ba shba s
sudo vim /etc/ssh/ssh_config

//在最后俩行加上

Host *
          SendEnv LANG LC_*
          KeepAlive yes         //主要是这行
```

