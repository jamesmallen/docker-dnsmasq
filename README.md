# docker-dnsmasq

This is a fork of [andyshinn/dnsmasq] that removes the need for `NET_ADMIN` capabilities.

It's a [dnsmasq][dnsmasq] Docker image. It is only 6 MB in size. It is just an `ENTRYPOINT` to the `dnsmasq` binary. Can you smell what the rock is cookin'?

## Usage

Available tags include:

* `jamesmallen/dnsmasq:latest`: same as 2.78
* `jamesmallen/dnsmasq:2.78`: dnsmasq 2.78 based on Alpine 3.7

Start the image with `docker run -p 53:53/tcp -p 53:53/udp jamesmallen/dnsmasq`.

The configuration is all handled on the command line (no wrapper scripts here). The `ENTRYPOINT` is `dnsmasq --keep-in-foreground --user=root` to keep it running in the foreground. If you wanted to send requests for an internal domain (such as Consul) you can forward the requests upstream using something like `docker run -p 53:53/tcp -p 53:53/udp jamesmallen/dnsmasq --address=/consul/10.17.0.2`. This will send a request for `redis.service.consul` to `10.17.0.2`.

As this is a very barebones entrypoint with just enough to run in the foreground, there is no logging enabled by default. To send logging to stdout you can add `--log-facility=-` as an option.

## Advanced usage

You can run multiple containers with this image if you don't force the port to `53` on the host. Here is a script that can be used to dynamically start a container that will run alongside other `dnsmasq` instances and generate a `resolver` file for use on Mac OSX:

```bash
#!/usr/bin/env bash
set -e

DNS_SUFFIX=dev.localhost

instance_id=$(docker run -p 53 -p 53/udp jamesmallen/dnsmasq --address=/${DNS_SUFFIX}/127.0.0.1)
dnsmasq_port=docker inspect --format='{{(index (index .NetworkSettings.Ports "53/udp") 0).HostPort}}' $instance_id
resolver_file="/etc/resolver/${DNS_SUFFIX}"
resolver_contents="$(cat << EOF
domain ${DNS_SUFFIX}
port ${dnsmasq_port}
nameserver 127.0.0.1.${dnsmasq_port}
EOF
)"

if [[ ! -f ${resolver_file} || $(< ${resolver_file}) != "${resolver_contents}" ]]; then
    echo "Updating ${resolver_file} (you may need to enter your password)..."
    sudo echo
    sudo mkdir -p "/etc/resolver"
    if [ -f "${resolver_file}" ]; then
        sudo rm "${resolver_file}"
    fi
    echo "${resolver_contents}" | sudo tee "${resolver_file}"
fi

```

[andyshinn/dnsmasq]: https://github.com/andyshinn/docker-dnsmasq
[dnsmasq]: http://www.thekelleys.org.uk/dnsmasq/doc.html
