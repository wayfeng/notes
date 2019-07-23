# Backup HOME folder with rsync

List of ignored files as 
``` bash
cat <<EOF > /tmp/ignore_list.txt
.aws
.bash_logout
.cache
.config
.dbus
.debug
.docker
.gnupg
.ICEauthority
.ipython
.keras
.local
.mozilla
.pki
.python*
.texlive*
.thumb*
.vim
.viminfo
.vnc/
.vccode/
.wget*
.X*
.x*
Desktop
Downloads
EOF
```

Run rsync
``` bash
rsync -aP --exclude-from=/tmp/ignore_list.txt /home/wayne /media/wayne/udisk/home_backup
```

