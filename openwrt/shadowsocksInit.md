## 不使用luci界面，从shadowsocks-openwrt的初始化脚本中提炼出来的启动脚本
## [URL](https://gist.github.com/stackia/0c83f9ca66cdc010be48840ee5b0a09c)
### shadowsocks
    #!/bin/sh /etc/rc.common

    START=90
    STOP=15

    NAME=shadowsocks
    EXTRA_COMMANDS=rules
    CONFIG_FILE=/etc/$NAME.json

    DELAY=10

    RULES_REMOTE_SERVER_IP=47.90.55.99
    RULES_BYPASS_LIST_FILE=/etc/chinadns_chnroute.txt

    REDIR_LOCAL_PORT=10086
    REDIR_MTU=1492

    LOCAL_LOCAL_PORT=10087
    LOCAL_MTU=1492

    TUNNEL_LOCAL_PORT=10088
    TUNNEL_DESTINATION=8.8.4.4:53
    TUNNEL_MTU=1492

    start_rules() {
      /data/ss-rules \
        -s $RULES_REMOTE_SERVER_IP \
        -l $REDIR_LOCAL_PORT \
        -B $RULES_BYPASS_LIST_FILE
    }

    rules() {
      if !(pidof ss-redir >/dev/null); then
        logger -st $NAME -p3 "ss-redir not running."
        return 1
      fi
      start_rules || /usr/bin/ss-rules -f
    }

    ss_redir() {
      command -v ss-redir >/dev/null 2>&1 || return 1
      ss-redir -c $CONFIG_FILE -u \
        -l $REDIR_LOCAL_PORT \
        --mtu $REDIR_MTU \
        -f /var/run/ss-redir.pid
    }

    ss_local() {
      command -v ss-local >/dev/null 2>&1 || return 0
      ss-local -c $CONFIG_FILE -u \
        -l $LOCAL_LOCAL_PORT \
        --mtu $LOCAL_MTU \
        -f /var/run/ss-local.pid
    }

    ss_tunnel() {
      command -v ss-tunnel >/dev/null 2>&1 || return 0
      ss-tunnel -c $CONFIG_FILE -u \
        -l $TUNNEL_LOCAL_PORT \
        -L $TUNNEL_DESTINATION \
        --mtu $TUNNEL_MTU \
        -f /var/run/ss-tunnel.pid
    }

    start() {
      mkdir -p /var/run /var/etc
      ss_redir && rules
      ss_local
      ss_tunnel
    }

    delay_start() {
      (sleep $1 && start >/dev/null 2>&1) &
    }

    boot() {
      if [ "$DELAY" -gt 0 ]; then
        delay_start $DELAY
      else
        start
      fi
      return 0
    }

    kill_all() {
      kill -9 $(pidof $@) >/dev/null 2>&1
    }

    stop() {
      /data/ss-rules -f
      kill_all ss-redir ss-local ss-tunnel
      if [ -f /var/run/ss-plugin ]; then
        kill -9 $(pidof $(cat /var/run/ss-plugin | sort | uniq)) >/dev/null 2>&1
        rm -f /var/run/ss-plugin
      fi
    }


### ss-rules
    #!/bin/sh
    #
    # Copyright (C) 2014-2017 Jian Chang <aa65535@live.com>
    #
    # This is free software, licensed under the GNU General Public License v3.
    # See /LICENSE for more information.
    #

    usage() {
      cat <<-EOF
        Usage: ss-rules [options]
        Valid options are:
            -s <server_ips>         ip address of shadowsocks remote server
            -l <local_port>         port number of shadowsocks local server
            -S <server_ips>         ip address of shadowsocks remote UDP server
            -L <local_port>         port number of shadowsocks local UDP server
            -B <ip_list_file>       a file whose content is bypassed ip list
            -b <wan_ips>            wan ip of will be bypassed
            -W <ip_list_file>       a file whose content is forwarded ip list
            -w <wan_ips>            wan ip of will be forwarded
            -I <interface>          proxy only for the given interface
            -d <target>             the default target of lan access control
            -a <lan_hosts>          lan ip of access control, need a prefix to
                                    define proxy type
            -e <extra_args>         extra arguments for iptables
            -o                      apply the rules to the OUTPUT chain
            -O                      apply the global rules to the OUTPUT chain
            -u                      enable udprelay mode, TPROXY is required
            -U                      enable udprelay mode, using different IP
                                    and ports for TCP and UDP
            -f                      flush the rules
            -h                      show this help message and exit
    EOF
      exit $1
    }

    loger() {
      # 1.alert 2.crit 3.err 4.warn 5.notice 6.info 7.debug
      logger -st ss-rules[$$] -p$1 $2
    }

    flush_rules() {
      iptables-save -c | grep -v "SS_SPEC" | iptables-restore -c
      if command -v ip >/dev/null 2>&1; then
        ip rule del fwmark 1 lookup 100 2>/dev/null
        ip route del local default dev lo table 100 2>/dev/null
      fi
      for setname in $(ipset -n list | grep "ss_spec"); do
        ipset destroy $setname 2>/dev/null
      done
      return 0
    }

    ipset_init() {
      ipset -! restore <<-EOF || return 1
        create ss_spec_src_ac hash:ip hashsize 64
        create ss_spec_src_bp hash:ip hashsize 64
        create ss_spec_src_fw hash:ip hashsize 64
        create ss_spec_dst_sp hash:net hashsize 64
        create ss_spec_dst_bp hash:net hashsize 64
        create ss_spec_dst_fw hash:net hashsize 64
        $(gen_lan_host_ipset_entry)
        $(gen_special_purpose_ip | sed -e "s/^/add ss_spec_dst_sp /")
        $(sed -e "s/^/add ss_spec_dst_bp /" ${WAN_BP_LIST:=/dev/null} 2>/dev/null)
        $(for ip in $WAN_BP_IP; do echo "add ss_spec_dst_bp $ip"; done)
        $(sed -e "s/^/add ss_spec_dst_fw /" ${WAN_FW_LIST:=/dev/null} 2>/dev/null)
        $(for ip in $WAN_FW_IP; do echo "add ss_spec_dst_fw $ip"; done)
    EOF
      return 0
    }

    ipt_nat() {
      include_ac_rules nat
      ipt="iptables -t nat"
      $ipt -A SS_SPEC_WAN_FW -p tcp \
        -j REDIRECT --to-ports $local_port || return 1
      if [ -n "$OUTPUT" ]; then
        $ipt -N SS_SPEC_WAN_DG
        $ipt -A SS_SPEC_WAN_DG -m set --match-set ss_spec_dst_sp dst -j RETURN
        $ipt -A SS_SPEC_WAN_DG -p tcp $EXT_ARGS -j $OUTPUT
        $ipt -I OUTPUT 1 -p tcp -j SS_SPEC_WAN_DG
      fi
      return $?
    }

    ipt_mangle() {
      [ -n "$TPROXY" ] || return 0
      if !(lsmod | grep -q TPROXY && command -v ip >/dev/null); then
        loger 4 "TPROXY or ip not found."
        return 0
      fi
      ip rule add fwmark 1 lookup 100
      ip route add local default dev lo table 100
      include_ac_rules mangle
      iptables -t mangle -A SS_SPEC_WAN_FW -p udp \
        -j TPROXY --on-port $LOCAL_PORT --tproxy-mark 0x01/0x01
      return $?
    }

    export_ipt_rules() {
      [ -n "$FWI" ] || return 0
      cat <<-CAT >>$FWI
      iptables-restore -n <<-EOF
      $(iptables-save | grep -E "SS_SPEC|^\*|^COMMIT" |\
          sed -e "s/^-A \(OUTPUT\|PREROUTING\)/-I \1 1/")
      EOF
    CAT
      return $?
    }

    gen_lan_host_ipset_entry() {
      for host in $LAN_HOSTS; do
        case "${host:0:1}" in
          n|N)
            echo add ss_spec_src_ac ${host:2}
            ;;
          b|B)
            echo add ss_spec_src_bp ${host:2}
            ;;
          g|G)
            echo add ss_spec_src_fw ${host:2}
            ;;
        esac
      done
    }

    gen_special_purpose_ip() {
      cat <<-EOF | grep -E "^([0-9]{1,3}\.){3}[0-9]{1,3}"
        0.0.0.0/8
        10.0.0.0/8
        100.64.0.0/10
        127.0.0.0/8
        169.254.0.0/16
        172.16.0.0/12
        192.0.0.0/24
        192.0.2.0/24
        192.31.196.0/24
        192.52.193.0/24
        192.88.99.0/24
        192.168.0.0/16
        192.175.48.0/24
        198.18.0.0/15
        198.51.100.0/24
        203.0.113.0/24
        224.0.0.0/4
        240.0.0.0/4
        255.255.255.255
        $server
        $SERVER
    EOF
    }

    include_ac_rules() {
      local protocol=$([ "$1" = "mangle" ] && echo udp || echo tcp)
      iptables-restore -n <<-EOF
      *$1
      :SS_SPEC_LAN_DG - [0:0]
      :SS_SPEC_LAN_AC - [0:0]
      :SS_SPEC_WAN_AC - [0:0]
      :SS_SPEC_WAN_FW - [0:0]
      -A SS_SPEC_LAN_DG -m set --match-set ss_spec_dst_sp dst -j RETURN
      -A SS_SPEC_LAN_DG -p $protocol $EXT_ARGS -j SS_SPEC_LAN_AC
      -A SS_SPEC_LAN_AC -m set --match-set ss_spec_src_bp src -j RETURN
      -A SS_SPEC_LAN_AC -m set --match-set ss_spec_src_fw src -j SS_SPEC_WAN_FW
      -A SS_SPEC_LAN_AC -m set --match-set ss_spec_src_ac src -j SS_SPEC_WAN_AC
      -A SS_SPEC_LAN_AC -j ${LAN_TARGET:=SS_SPEC_WAN_AC}
      -A SS_SPEC_WAN_AC -m set --match-set ss_spec_dst_fw dst -j SS_SPEC_WAN_FW
      -A SS_SPEC_WAN_AC -m set --match-set ss_spec_dst_bp dst -j RETURN
      -A SS_SPEC_WAN_AC -j SS_SPEC_WAN_FW
      $(gen_prerouting_rules $protocol)
      COMMIT
    EOF
    }

    gen_prerouting_rules() {
      [ -z "$IFNAMES" ] && echo -I PREROUTING 1 -p $1 -j SS_SPEC_LAN_DG
      for ifname in $IFNAMES; do
        echo -I PREROUTING 1 -i $ifname -p $1 -j SS_SPEC_LAN_DG
      done
    }

    while getopts ":s:l:S:L:B:b:W:w:I:d:a:e:oOuUfh" arg; do
      case "$arg" in
        s)
          server=$(for ip in $OPTARG; do echo $ip; done)
          ;;
        l)
          local_port=$OPTARG
          ;;
        S)
          SERVER=$(for ip in $OPTARG; do echo $ip; done)
          ;;
        L)
          LOCAL_PORT=$OPTARG
          ;;
        B)
          WAN_BP_LIST=$OPTARG
          ;;
        b)
          WAN_BP_IP=$OPTARG
          ;;
        W)
          WAN_FW_LIST=$OPTARG
          ;;
        w)
          WAN_FW_IP=$OPTARG
          ;;
        I)
          IFNAMES=$OPTARG
          ;;
        d)
          LAN_TARGET=$OPTARG
          ;;
        a)
          LAN_HOSTS=$OPTARG
          ;;
        e)
          EXT_ARGS=$OPTARG
          ;;
        o)
          OUTPUT=SS_SPEC_WAN_AC
          ;;
        O)
          OUTPUT=SS_SPEC_WAN_FW
          ;;
        u)
          TPROXY=1
          ;;
        U)
          TPROXY=2
          ;;
        f)
          flush_rules
          exit 0
          ;;
        h)
          usage 0
          ;;
      esac
    done

    [ -z "$server" -o -z "$local_port" ] && usage 2

    if [ "$TPROXY" = 1 ]; then
      unset SERVER
      LOCAL_PORT=$local_port
    elif [ "$TPROXY" = 2 ]; then
      : ${SERVER:?"You must assign an ip for the udp relay server."}
      : ${LOCAL_PORT:?"You must assign a port for the udp relay server."}
    fi

    flush_rules && ipset_init && ipt_nat && ipt_mangle && export_ipt_rules
    RET=$?
    [ "$RET" = 0 ] || loger 3 "Start failed!"
    exit $RET
