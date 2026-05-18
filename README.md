<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Sistem Kasir Supermart</title>
    <script src="https://unpkg.com/html5-qrcode"></script>
    <style>
        * { box-sizing: border-box; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        body { margin: 0; background-color: #eef2f5; display: flex; height: 100vh; overflow: hidden; }
        :root { --primary: #005bb5; --secondary: #ffde00; --danger: #dc3545; --success: #28a745; --dark: #343a40; }

        .left-panel { width: 65%; padding: 20px; overflow-y: auto; background: #f8f9fa; }
        .right-panel { width: 35%; background: white; border-left: 2px solid #ddd; display: flex; flex-direction: column; box-shadow: -2px 0 5px rgba(0,0,0,0.1); }

        .header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; background: var(--primary); color: white; padding: 15px; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        .header h1 { margin: 0; font-size: 24px; color: var(--secondary); }
        
        .input-group { display: flex; gap: 10px; }
        .barcode-input { padding: 10px; font-size: 16px; border-radius: 5px; border: 1px solid #ccc; width: 220px; }
        .btn-scan { padding: 10px 15px; background: var(--secondary); color: var(--dark); border: none; border-radius: 5px; font-weight: bold; cursor: pointer; }
        .btn-scan:hover { background: #e6c800; }

        /* Kotak Kamera */
        #reader-container { display: none; margin-bottom: 20px; background: white; padding: 10px; border-radius: 8px; border: 2px solid var(--primary); }
        #reader { width: 100%; max-width: 400px; margin: auto; }
        .btn-close-cam { background: var(--danger); color: white; border: none; padding: 8px; width: 100%; margin-top: 10px; cursor: pointer; border-radius: 5px; font-weight: bold; }

        .product-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(160px, 1px)); gap: 15px; }
        .product-card { background: white; border-radius: 8px; padding: 15px; text-align: center; cursor: pointer; transition: 0.2s; border: 1px solid #ddd; box-shadow: 0 2px 4px rgba(0,0,0,0.05); }
        .product-card:hover { transform: translateY(-3px); border-color: var(--primary); box-shadow: 0 4px 8px rgba(0,123,255,0.2); }
        .product-card h3 { font-size: 16px; margin: 10px 0 5px; color: var(--dark); }
        .product-card p { color: var(--primary); font-weight: bold; margin: 0; font-size: 18px; }
        .product-card small { color: #888; font-size: 12px; }

        .cart-header { background: var(--dark); color: white; padding: 20px; text-align: center; }
        .cart-header h2 { margin: 0; font-size: 20px; }
        
        .cart-items { flex-grow: 1; overflow-y: auto; padding: 10px; }
        .cart-item { display: flex; justify-content: space-between; align-items: center; padding: 10px; border-bottom: 1px solid #eee; }
        .cart-item-info { width: 60%; }
        .cart-item-name { font-weight: bold; font-size: 14px; margin-bottom: 5px; }
        .cart-item-qty { font-size: 12px; color: #666; }
        .cart-item-total { font-weight: bold; color: var(--primary); }
        .btn-del { background: var(--danger); color: white; border: none; padding: 5px 10px; border-radius: 4px; cursor: pointer; }

        .cart-summary { padding: 20px; background: #fdfdfd; border-top: 1px solid #ddd; }
        .summary-line { display: flex; justify-content: space-between; margin-bottom: 10px; font-size: 16px; }
        .summary-total { font-size: 22px; font-weight: bold; color: var(--danger); border-top: 2px dashed #ccc; padding-top: 10px; margin-top: 10px; }
        
        .payment-input { width: 100%; padding: 12px; font-size: 18px; font-weight: bold; margin: 10px 0; border: 2px solid var(--primary); border-radius: 5px; text-align: right; }
        
        .btn-pay { width: 100%; padding: 15px; font-size: 18px; font-weight: bold; background: var(--success); color: white; border: none; border-radius: 5px; cursor: pointer; transition: 0.2s; }
        .btn-pay:hover { background: #218838; }
    </style>
</head>
<body>

    <div class="left-panel">
        <div class="header">
            <div>
                <h1>🛒 SUPERMART POS</h1>
                <small>Sistem Kasir + Kamera Scanner</small>
            </div>
            <div class="input-group">
                <input type="text" id="barcodeInput" class="barcode-input" placeholder="Ketik Kode (Enter)..." autofocus>
                <button class="btn-scan" onclick="bukaKamera()">📷 Scan</button>
            </div>
        </div>

        <div id="reader-container">
            <div id="reader"></div>
            <button class="btn-close-cam" onclick="tutupKamera()">Tutup Kamera</button>
        </div>

        <div class="product-grid" id="productGrid"></div>
    </div>

    <div class="right-panel">
        <div class="cart-header"><h2>Rincian Transaksi</h2></div>
        <div class="cart-items" id="cartItems"><div style="text-align: center; color: #999; margin-top: 20px;">Belum ada barang</div></div>
        <div class="cart-summary">
            <div class="summary-line"><span>Subtotal</span><span id="subtotalText">Rp 0</span></div>
            <div class="summary-line"><span>PPN (11%)</span><span id="ppnText">Rp 0</span></div>
            <div class="summary-line summary-total"><span>TOTAL</span><span id="totalText" data-total="0">Rp 0</span></div>
            <input type="number" id="paymentInput" class="payment-input" placeholder="Masukkan Uang Tunai (Rp)">
            <button class="btn-pay" onclick="prosesPembayaran()">BAYAR & CETAK STRUK</button>
        </div>
    </div>

    <script>
        const databaseProduk = [
            // Tambahkan data barcode asli di bagian 'kode' kalau mau ditest scan langsung!
            { kode: "89686010023", nama: "Aqua Botol 600ml", harga: 3500, stok: 50 },
            { kode: "08968604321", nama: "Indomie Goreng", harga: 3500, stok: 100 },
            { kode: "003", nama: "Susu Bear Brand", harga: 10500, stok: 30 },
            { kode: "004", nama: "Roti Aoka Coklat", harga: 3000, stok: 40 }
        ];

        let keranjang = [];
        let html5QrcodeScanner = null;

        function formatRupiah(angka) {
            return new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', minimumFractionDigits: 0 }).format(angka);
        }

        function renderProduk() {
            const grid = document.getElementById('productGrid');
            grid.innerHTML = '';
            databaseProduk.forEach(produk => {
                const card = document.createElement('div');
                card.className = 'product-card';
                card.onclick = () => tambahKeKeranjang(produk.kode);
                card.innerHTML = `<h3>${produk.nama}</h3><p>${formatRupiah(produk.harga)}</p><small>Kode: ${produk.kode}</small>`;
                grid.appendChild(card);
            });
        }

        function tambahKeKeranjang(kode) {
            const produk = databaseProduk.find(p => p.kode === kode);
            if (!produk) { alert("Produk dengan kode " + kode + " tidak ditemukan!"); return; }

            const item = keranjang.find(i => i.kode === kode);
            if (item) {
                item.qty += 1;
                item.subtotal = item.qty * item.harga;
            } else {
                keranjang.push({ kode: produk.kode, nama: produk.nama, harga: produk.harga, qty: 1, subtotal: produk.harga });
            }
            updateUIKeranjang();
        }

        function hapusDariKeranjang(kode) {
            keranjang = keranjang.filter(item => item.kode !== kode);
            updateUIKeranjang();
        }

        function updateUIKeranjang() {
            const container = document.getElementById('cartItems');
            container.innerHTML = '';
            let subtotal = 0;

            if (keranjang.length === 0) {
                container.innerHTML = '<div style="text-align: center; color: #999; margin-top: 20px;">Belum ada barang</div>';
            } else {
                keranjang.forEach(item => {
                    subtotal += item.subtotal;
                    container.innerHTML += `
                        <div class="cart-item">
                            <div class="cart-item-info">
                                <div class="cart-item-name">${item.nama}</div>
                                <div class="cart-item-qty">${item.qty} x ${formatRupiah(item.harga)}</div>
                            </div>
                            <div class="cart-item-total">${formatRupiah(item.subtotal)}</div>
                            <button class="btn-del" onclick="hapusDariKeranjang('${item.kode}')">X</button>
                        </div>
                    `;
                });
            }

            const ppn = subtotal * 0.11;
            const total = subtotal + ppn;

            document.getElementById('subtotalText').innerText = formatRupiah(subtotal);
            document.getElementById('ppnText').innerText = formatRupiah(ppn);
            document.getElementById('totalText').innerText = formatRupiah(total);
            document.getElementById('totalText').setAttribute('data-total', total);
        }

        document.getElementById('barcodeInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter' && this.value.trim() !== "") {
                tambahKeKeranjang(this.value.trim());
                this.value = ''; 
            }
        });

        // --- FITUR KAMERA SCANNER BARCODE ---
        function bukaKamera() {
            document.getElementById('reader-container').style.display = 'block';
            if (!html5QrcodeScanner) {
                html5QrcodeScanner = new Html5Qrcode("reader");
            }
            html5QrcodeScanner.start(
                { facingMode: "environment" }, // Pakai kamera belakang (kalau ada)
                { fps: 10, qrbox: {width: 250, height: 150} },
                (decodedText) => {
                    // Berhasil scan!
                    tutupKamera();
                    tambahKeKeranjang(decodedText);
                    // Bunyi Beep Kasir
                    let audio = new Audio('https://www.soundjay.com/buttons/beep-07.wav');
                    audio.play();
                },
                (errorMessage) => { /* Abaikan error saat proses mencari barcode */ }
            ).catch(err => {
                alert("Gagal membuka kamera: " + err);
                tutupKamera();
            });
        }

        function tutupKamera() {
            if (html5QrcodeScanner) {
                html5QrcodeScanner.stop().then(() => {
                    document.getElementById('reader-container').style.display = 'none';
                }).catch(err => console.log(err));
            } else {
                document.getElementById('reader-container').style.display = 'none';
            }
        }

        function prosesPembayaran() {
            if (keranjang.length === 0) { alert("Keranjang kosong!"); return; }
            const total = parseFloat(document.getElementById('totalText').getAttribute('data-total'));
            const bayar = parseFloat(document.getElementById('paymentInput').value);
            if (isNaN(bayar) || bayar < total) { alert("Uang kurang!"); return; }
            alert("Pembayaran Berhasil! Kembalian: " + formatRupiah(bayar - total) + "\n\n(Struk akan di-print otomatis)");
            keranjang = []; document.getElementById('paymentInput').value = ''; updateUIKeranjang();
        }

        window.onload = function() { renderProduk(); document.getElementById('barcodeInput').focus(); };
    </script>
</body>
</html>
