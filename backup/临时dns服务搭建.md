通过coredns镜像，nerdctl run启动dns服务器，监听53端口，
启动dns：`bash run-tmp-dns.sh -d lyx.nginx.com=10.44.70.210,2003:db8::11 -d test.com=1.1.1.1`
卸载：`bash run-tmp-dns.sh -u `
```
#!/bin/bash
set -e

IMAGE="registry.kylincloud.org:4001/kcc/kcc/coredns:v1.8.6"
CONTAINER_NAME="temp-coredns"
WORKDIR="/opt/temp-coredns"
DNS_PORT=53

usage() {
cat <<EOF
Usage:

Start DNS:
  $0 -d domain=ip[,ip] [-d domain=ip]

Uninstall:
  $0 -u

Example:
  $0 -d test1.com=10.42.1.10,2001:db8::10 -d test2.com=10.42.1.20
  $0 -u
EOF
exit 1
}

uninstall() {
    echo "remove container ${CONTAINER_NAME}"
    nerdctl rm -f ${CONTAINER_NAME} >/dev/null 2>&1 || true
    echo "remove config ${WORKDIR}"
    rm -rf ${WORKDIR}
    echo "done"
    exit 0
}

declare -a DOMAINS

while getopts "d:u" opt; do
    case $opt in
        d)
            DOMAINS+=("$OPTARG")
            ;;
        u)
            uninstall
            ;;
        *)
            usage
            ;;
    esac
done

if [ ${#DOMAINS[@]} -eq 0 ]; then
    usage
fi

mkdir -p ${WORKDIR}

cat > ${WORKDIR}/Corefile <<EOF
.:53 {
    errors
    log
    hosts {
EOF

for item in "${DOMAINS[@]}"; do
    domain=$(echo "$item" | cut -d= -f1)
    ips=$(echo "$item" | cut -d= -f2)

    IFS=',' read -ra IP_ARRAY <<< "$ips"

    for ip in "${IP_ARRAY[@]}"; do
        echo "        ${ip} ${domain}" >> ${WORKDIR}/Corefile
    done
done

cat >> ${WORKDIR}/Corefile <<EOF
        fallthrough
    }
    cache 30
}
EOF

echo "Corefile:"
cat ${WORKDIR}/Corefile

nerdctl rm -f ${CONTAINER_NAME} >/dev/null 2>&1 || true

nerdctl run -d \
    --name ${CONTAINER_NAME} \
    --restart always \
    -p ${DNS_PORT}:53/udp \
    -p ${DNS_PORT}:53/tcp \
    -v ${WORKDIR}/Corefile:/etc/coredns/Corefile:ro \
    ${IMAGE} \
    -conf /etc/coredns/Corefile

NODE_IP=$(hostname -I | awk '{print $1}')

echo
echo "DNS started: ${NODE_IP}:${DNS_PORT}"
echo "Test:"
echo "  dig @${NODE_IP} domain A"
echo "  dig @${NODE_IP} domain AAAA"
```