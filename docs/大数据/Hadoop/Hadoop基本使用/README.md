# hadoop基本使用

xcall.sh脚本编写

```shell
#!/bin/sh
pcount=$#
if((pcount==0));then
        echo no args...;
        exit;
fi
echo ==================master==================
$@
for((host=1; host<=2; host++)); do
        echo ==================slave$host==================
        ssh slave$host "source /etc/profile;$@"
done
```

## xcp.sh

```shell
#!/bin/bash
if [ $# -lt 1 ] ;then
  echo no args
  exit;
fi
 
#get first argument
arg1=$1;                          #qu chu di yi ge can shu
cuser=`whoami`                    #qu chu yong hu shi shei
fname=`basename $arg1`            #qu chu weng jian ming
dir=`dirname $arg1`               #
if [ $dir="." ] ;then
   dir=`pwd`
fi
for (( i=1;i<=4;i=i+1 )) ;
do
  echo ---------- coping $arg1 to ubuntu$i ---------- ;
  if [ -d $arg1 ] ;then
    scp -r $arg1 $cuser@ubuntu$i:$dir
  else
    scp $arg1 $cuser@ubuntu$i:$dir
  fi
done
```

# xrm.sh

```shell

#!/bin/bash
if [ $# -lt 1 ] ;then
  echo no args
  exit;
fi
 
#get first argument
arg1=$1;                          #qu chu di yi ge can shu
cuser=`whoami`                    #qu chu yong hu shi shei
fname=`basename $arg1`            #qu chu weng jian ming
dir=`dirname $arg1`               #
if [ $dir="." ] ;then
   dir=`pwd`
fi
 
echo ------ rming $arg1 from localhost ----
rm -rf $arg1
echo
 
for (( i=1;i<=4;i=i+1 )) ;
do
  echo ---------- rming $arg1 from ubuntu$i ---------- ;
  ssh ubuntu$i rm -rf $dir/$fname
  echo
done
```

