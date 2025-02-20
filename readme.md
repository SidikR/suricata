# Konfigurasi Suricata dengan NFQUEUE dan Iptables

## 1. Ubah Konfigurasi Suricata
Buka file konfigurasi Suricata:
```bash
nano /etc/default/suricata
```
Ganti baris berikut:
```plaintext
LISTENMODE=nfqueue
```

## 2. Buat Skrip Kontrol Suricata
Buat file `suricata_control.sh`:
```bash
sudo su
cd /usr/local/bin
nano suricata_control.sh
```

Isi dengan skrip berikut:
```bash
#!/bin/bash

# Fungsi untuk menghentikan Suricata dan menghapus aturan iptables
stop_suricata() {
    echo "Menghentikan Suricata..."
    systemctl stop suricata

    if pgrep -x "suricata" > /dev/null; then
        echo "Suricata masih berjalan, mencoba mematikannya secara paksa..."
        killall suricata
    fi

    iptables -D INPUT -j NFQUEUE --queue-num 0 2>/dev/null
    iptables -D OUTPUT -j NFQUEUE --queue-num 0 2>/dev/null
    iptables -D FORWARD -j NFQUEUE --queue-num 0 2>/dev/null

    echo "Suricata dihentikan dan aturan NFQUEUE dihapus."
}

# Fungsi untuk memulai Suricata dan menambahkan aturan iptables
start_suricata() {
    echo "Memulai Suricata..."
    systemctl start suricata

    if systemctl is-active --quiet suricata; then
        echo "Suricata berhasil dimulai."
        iptables -A INPUT -j NFQUEUE --queue-num 0
        iptables -A OUTPUT -j NFQUEUE --queue-num 0
        iptables -A FORWARD -j NFQUEUE --queue-num 0
        echo "Aturan NFQUEUE ditambahkan."
    else
        echo "Gagal memulai Suricata."
    fi
}

# Fungsi untuk restart Suricata
restart_suricata() {
    echo "Merestart Suricata..."
    stop_suricata
    start_suricata
}

# Menu pilihan untuk pengguna
echo "Pilih tindakan yang ingin dilakukan:"
echo "1) Stop Suricata"
echo "2) Start Suricata"
echo "3) Restart Suricata"
read -p "Masukkan pilihan (1/2/3): " pilihan

case $pilihan in
    1)
        stop_suricata
        ;;
    2)
        start_suricata
        ;;
    3)
        restart_suricata
        ;;
    *)
        echo "Pilihan tidak valid."
        ;;
esac
```

Beri izin eksekusi pada skrip:
```bash
chmod +x suricata_control.sh
```
Jalankan skrip dengan:
```bash
./suricata_control.sh
```

## 3. Konfigurasi `suricata.yaml`
Buka file konfigurasi:
```bash
nano /etc/suricata/suricata.yaml
```
Sesuaikan pengaturan berikut:
```yaml
af-packet:
  - interface: enp0s3 # Sesuaikan dengan interface yang digunakan
    threads: auto
    cluster-id: 99
    cluster-type: cluster_flow
    nfqueue: 0
    mode: inline
```

## 4. Konfigurasi Iptables
Flush aturan lama:

Kalo kamu ngelakuin ini ssh akan teroputus cuy, jadi lakuin di server aja

```bash
iptables -F
iptables -X
```

Set izin SSH agar tidak diarahkan ke Suricata:
```bash
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT
```

Tambahkan aturan NFQUEUE:
```bash
sudo iptables -A INPUT -j NFQUEUE --queue-num 0
sudo iptables -A OUTPUT -j NFQUEUE --queue-num 0
sudo iptables -A FORWARD -j NFQUEUE --queue-num 0
```

Jika ingin menyimpan aturan ini agar tetap berlaku setelah reboot, jalankan:
```bash
sudo iptables-save | sudo tee /etc/iptables/rules.v4
sudo apt update && sudo apt install iptables-persistent -y
```

