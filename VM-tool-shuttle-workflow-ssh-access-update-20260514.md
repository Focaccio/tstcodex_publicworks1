# VM Tool Shuttle SSH Access Update - 2026-05-14

This update records the SSH-access changes learned during `vm-shuttle-test01`. Apply these requirements to future workflow runs.

## Required result

A future test is not complete until the VM is reachable on the LAN and SSH login works.

## Required guest packages

```bash
sudo apt update
sudo apt install -y openssh-server qemu-guest-agent open-vm-tools cloud-init
```

## Required SSH service setup

```bash
sudo mkdir -p /run/sshd
sudo install -d -m 0755 /etc/ssh/sshd_config.d
printf 'PasswordAuthentication yes\nKbdInteractiveAuthentication yes\n' | sudo tee /etc/ssh/sshd_config.d/99-codex-password-auth.conf
sudo sshd -t
sudo systemctl enable --now ssh
sudo systemctl enable --now qemu-guest-agent
sudo systemctl enable --now open-vm-tools || true
```

## Required validation

```bash
virsh domifaddr VM_NAME --source agent
# Ignore 127.0.0.1 and record the non-loopback LAN IP.
ping -c 3 VM_LAN_IP
nc -vz -w 2 VM_LAN_IP 22
ssh greg@VM_LAN_IP 'hostname; ip -br addr; id; systemctl is-active ssh qemu-guest-agent open-vm-tools'
```

## Known failure from test 1

The first SSH repair attempt wrote a literal `\n` into `/etc/ssh/sshd_config.d/99-codex-password-auth.conf`, causing sshd to reject the drop-in and refuse connections. The file must contain real separate lines.

## IP parsing requirement

When parsing `virsh domifaddr`, do not accept loopback addresses. The successful bridged test showed the VM on `enp1s0` with `192.168.86.49/24`; the harness initially selected `127.0.0.1` and logged a false failure.
