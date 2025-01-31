# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

    config.vm.box = "opensuse/Tumbleweed.x86_64"
    config.vm.box_version = "1.0.20230819"

    config.vm.hostname = 'loki'
    config.vm.network :private_network, ip: '192.168.56.10'

    config.vm.provider "virtualbox" do |vb|
        vb.memory = 1024 * 16
        vb.cpus = 4
    end

    if Vagrant.has_plugin?("vagrant-proxyconf")
        # if required then configure the proxy
        config.proxy.enabled = false
    end

    config.vm.provision "shell", inline: $provision_privileged
    config.vm.provision "shell", inline: $provision, privileged: false

end

$provision_privileged = <<-SCRIPT
zypper refresh
zypper update
zypper --non-interactive install docker htop jq tree tmux unzip
zypper --non-interactive install git-core go1.20 make

systemctl enable docker
systemctl start docker
systemctl status docker

usermod -aG docker vagrant
SCRIPT

$provision = <<-SCRIPT
pushd /vagrant
test -e .env || ln -s dotenv .env || cp dotenv .env
popd

mkdir -p ~/{loki,prometheus,promtail/log}

#
# install Docker Compose
#
curl -L --silent https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-linux-x86_64 -o ~/.local/bin/docker-compose
chmod +x ~/.local/bin/docker-compose

#
# install Grafana logcli
#
curl -L --silent https://github.com/grafana/loki/releases/download/v2.8.4/logcli-linux-amd64.zip -o /tmp/logcli-linux-amd64.zip
unzip /tmp/logcli-linux-amd64.zip -d ~/.local/bin/
mv ~/.local/bin/logcli-linux-amd64 ~/.local/bin/logcli
chmod +x ~/.local/bin/logcli

#
# install Grafana xk6
#
GOBIN=~/.local/bin/ go install go.k6.io/xk6/cmd/xk6@v0.9.2

#
# build w/ loki k6 extension
#
mkdir -p src/github.com/grafana
git clone https://github.com/grafana/xk6-loki
pushd xk6-loki
git checkout -b c5c764b
make k6
mv k6 ~/.local/bin/k6
popd

#
# install NVM and Node.js
#
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.4/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install v18.17.0

cat << 'EOF' >> ~/.bashrc

export LOKI_ADDR=http://localhost:3100
eval "$(logcli --completion-script-bash)"
EOF

SCRIPT
