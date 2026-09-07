<?php
session_start();

/*
|--------------------------------------------------------------------------
| KONEKSI DATABASE
|--------------------------------------------------------------------------
| Sesuaikan dengan database kamu.
*/
$conn = mysqli_connect("localhost", "root", "", "kasir");

if (!$conn) {
    die("Koneksi database gagal: " . mysqli_connect_error());
}

/*
|--------------------------------------------------------------------------
| CART
|--------------------------------------------------------------------------
*/
if (!isset($_SESSION['cart'])) {
    $_SESSION['cart'] = [];
}

/*
|--------------------------------------------------------------------------
| TAMBAH PRODUK
|--------------------------------------------------------------------------
*/
if (isset($_POST['add_product'])) {

    $name = mysqli_real_escape_string($conn, $_POST['name']);
    $price = (int) $_POST['price'];
    $stock = (int) $_POST['stock'];
    $category = mysqli_real_escape_string($conn, $_POST['category']);
    $expired_date = $_POST['expired_date'];

    mysqli_query($conn, "
        INSERT INTO products
        (name, price, stock, category, expired_date)
        VALUES
        ('$name', '$price', '$stock', '$category', '$expired_date')
    ");

    header("Location: index.php?page=products");
    exit;
}

/*
|--------------------------------------------------------------------------
| HAPUS PRODUK
|--------------------------------------------------------------------------
*/
if (isset($_GET['delete_product'])) {

    $id = (int) $_GET['delete_product'];

    mysqli_query($conn, "DELETE FROM products WHERE id=$id");

    header("Location: index.php?page=products");
    exit;
}

/*
|--------------------------------------------------------------------------
| TAMBAH CUSTOMER
|--------------------------------------------------------------------------
*/
if (isset($_POST['add_customer'])) {

    $name = mysqli_real_escape_string($conn, $_POST['name']);
    $phone = mysqli_real_escape_string($conn, $_POST['phone']);
    $address = mysqli_real_escape_string($conn, $_POST['address']);
    $email = mysqli_real_escape_string($conn, $_POST['email']);

    mysqli_query($conn, "
        INSERT INTO customers
        (name, phone, address, email)
        VALUES
        ('$name', '$phone', '$address', '$email')
    ");

    header("Location: index.php?page=customers");
    exit;
}

/*
|--------------------------------------------------------------------------
| HAPUS CUSTOMER
|--------------------------------------------------------------------------
*/
if (isset($_GET['delete_customer'])) {

    $id = (int) $_GET['delete_customer'];

    mysqli_query($conn, "DELETE FROM customers WHERE id=$id");

    header("Location: index.php?page=customers");
    exit;
}

/*
|--------------------------------------------------------------------------
| TAMBAH KE KERANJANG
|--------------------------------------------------------------------------
*/
if (isset($_GET['add_cart'])) {

    $id = (int) $_GET['add_cart'];

    $result = mysqli_query(
        $conn,
        "SELECT * FROM products WHERE id=$id AND stock > 0"
    );

    $product = mysqli_fetch_assoc($result);

    if ($product) {

        $found = false;

        foreach ($_SESSION['cart'] as $index => $cart) {

            if ($cart['id'] == $id) {

                $_SESSION['cart'][$index]['qty']++;

                $found = true;
                break;
            }
        }

        if (!$found) {

            $_SESSION['cart'][] = [
                'id' => $product['id'],
                'name' => $product['name'],
                'price' => $product['price'],
                'qty' => 1
            ];
        }
    }

    header("Location: index.php?page=cashier");
    exit;
}

/*
|--------------------------------------------------------------------------
| HAPUS CART
|--------------------------------------------------------------------------
*/
if (isset($_GET['remove_cart'])) {

    $index = (int) $_GET['remove_cart'];

    if (isset($_SESSION['cart'][$index])) {
        unset($_SESSION['cart'][$index]);
        $_SESSION['cart'] = array_values($_SESSION['cart']);
    }

    header("Location: index.php?page=cashier");
    exit;
}

/*
|--------------------------------------------------------------------------
| CHECKOUT
|--------------------------------------------------------------------------
*/
if (isset($_POST['checkout'])) {

    $customer_name = mysqli_real_escape_string(
        $conn,
        $_POST['customer_name']
    );

    $pay = (int) $_POST['pay'];

    $total = 0;

    foreach ($_SESSION['cart'] as $cart) {
        $total += $cart['price'] * $cart['qty'];
    }

    if ($total > 0 && $pay >= $total) {

        $change_money = $pay - $total;

        $invoice = "INV-" . date("YmdHis");

        mysqli_query($conn, "
            INSERT INTO transactions
            (invoice, customer_name, total, pay, change_money, created_at)
            VALUES
            (
                '$invoice',
                '$customer_name',
                '$total',
                '$pay',
                '$change_money',
                NOW()
            )
        ");

        foreach ($_SESSION['cart'] as $cart) {

            $product_id = $cart['id'];
            $product_name = mysqli_real_escape_string(
                $conn,
                $cart['name']
            );

            $qty = $cart['qty'];
            $price = $cart['price'];
            $subtotal = $price * $qty;

            mysqli_query($conn, "
                INSERT INTO transaction_details
                (invoice, product_id, product_name, qty, price, subtotal)
                VALUES
                (
                    '$invoice',
                    '$product_id',
                    '$product_name',
                    '$qty',
                    '$price',
                    '$subtotal'
                )
            ");

            mysqli_query($conn, "
                UPDATE products
                SET stock = stock - $qty
                WHERE id = $product_id
            ");
        }

        $_SESSION['cart'] = [];
        $_SESSION['last_invoice'] = $invoice;

        header("Location: index.php?page=receipt");
        exit;

    } else {

        $error_checkout = "Jumlah pembayaran kurang atau keranjang masih kosong.";
    }
}

$page = isset($_GET['page']) ? $_GET['page'] : 'dashboard';

?>

<!DOCTYPE html>
<html>
<head>

    <title>Salad Saturday - Aplikasi Kasir</title>

    <meta charset="UTF-8">

    <meta
        name="viewport"
        content="width=device-width, initial-scale=1.0"
    >

    <style>

    *{
        margin:0;
        padding:0;
        box-sizing:border-box;
        font-family:Arial,sans-serif;
    }

    body{
        background:#0f172a;
        color:white;
    }

    .sidebar{
        width:250px;
        background:#1e293b;
        height:100vh;
        position:fixed;
        padding:20px;
        left:0;
        top:0;
    }

    .sidebar h2{
        margin-bottom:30px;
        text-align:center;
    }

    .sidebar a{
        display:block;
        color:white;
        text-decoration:none;
        padding:14px;
        margin-top:10px;
        border-radius:10px;
        background:#334155;
    }

    .sidebar a:hover{
        background:#3b82f6;
    }

    .main{
        margin-left:270px;
        padding:20px;
    }

    .topbar{
        width:100%;
        background:#1e293b;
        padding:20px;
        border-radius:20px;
        margin-bottom:20px;
    }

    .card-grid{
        display:grid;
        grid-template-columns:
        repeat(auto-fit,minmax(220px,1fr));

        gap:20px;
        margin-top:20px;
    }

    .card{
        background:#1e293b;
        padding:25px;
        border-radius:20px;
    }

    .card h2{
        font-size:32px;
        margin-top:10px;
    }

    .flex{
        display:flex;
        gap:20px;
    }

    .w-50{
        width:50%;
    }

    input,
    select,
    textarea{
        width:100%;
        padding:14px;
        border:none;
        border-radius:10px;
        margin-top:10px;
        background:#334155;
        color:white;
    }

    button{
        padding:12px 18px;
        border:none;
        border-radius:10px;
        background:#3b82f6;
        color:white;
        cursor:pointer;
        margin-top:10px;
    }

    button:hover{
        opacity:.85;
    }

    table{
        width:100%;
        border-collapse:collapse;
        margin-top:20px;
    }

    table th{
        background:#1e293b;
        padding:15px;
    }

    table td{
        padding:15px;
        background:#334155;
        text-align:center;
    }

    .product-card{
        background:#1e293b;
        padding:20px;
        border-radius:20px;
    }

    .grid-product{
        display:grid;
        grid-template-columns:
        repeat(auto-fit,minmax(220px,1fr));

        gap:20px;
    }

    .receipt{
        background:white;
        color:black;
        width:400px;
        max-width:100%;
        margin:auto;
        padding:20px;
        border-radius:10px;
    }

    .receipt h2{
        text-align:center;
        margin-bottom:20px;
    }

    .receipt table td{
        background:white;
        color:black;
    }

    .danger{
        background:#dc2626;
        padding:12px;
        border-radius:10px;
        margin-top:15px;
    }

    @media(max-width:768px){

        .sidebar{
            width:100%;
            height:auto;
            position:relative;
        }

        .main{
            margin-left:0;
        }

        .flex{
            flex-direction:column;
        }

        .w-50{
            width:100%;
        }

        .receipt{
            width:100%;
        }
    }

    @media print{

        body{
            background:white;
        }

        .sidebar,
        .topbar,
        .receipt button{
            display:none;
        }

        .main{
            margin:0;
            padding:0;
        }

        .receipt{
            box-shadow:none;
        }
    }

    </style>

</head>

<body>

<!-- =========================================================
     SIDEBAR
========================================================= -->

<div class="sidebar">

    <h2>🥗 SALAD SATURDAY</h2>

    <a href="index.php">
        🏠 Dashboard
    </a>

    <a href="index.php?page=products">
        📦 Produk
    </a>

    <a href="index.php?page=customers">
        👥 Customer
    </a>

    <a href="index.php?page=cashier">
        🛒 Kasir
    </a>

    <a href="index.php?page=transactions">
        🧾 Transaksi
    </a>

</div>

<!-- =========================================================
     MAIN
========================================================= -->

<div class="main">


<?php if($page == 'dashboard'): ?>

<div class="topbar">

    <h1>Dashboard</h1>

    <p style="margin-top:8px;">
        Selamat datang di Aplikasi Kasir Salad Saturday
    </p>

</div>

<?php

$totalProduct = mysqli_num_rows(
    mysqli_query($conn, "SELECT * FROM products")
);

$totalCustomer = mysqli_num_rows(
    mysqli_query($conn, "SELECT * FROM customers")
);

$totalTransaction = mysqli_num_rows(
    mysqli_query($conn, "SELECT * FROM transactions")
);

$getIncome = mysqli_query(
    $conn,
    "SELECT SUM(total) AS income FROM transactions"
);

$income = mysqli_fetch_assoc($getIncome);

?>

<div class="card-grid">

    <div class="card">

        <h3>Total Produk</h3>

        <h2>
            <?php echo $totalProduct; ?>
        </h2>

    </div>


    <div class="card">

        <h3>Total Customer</h3>

        <h2>
            <?php echo $totalCustomer; ?>
        </h2>

    </div>


    <div class="card">

        <h3>Total Transaksi</h3>

        <h2>
            <?php echo $totalTransaction; ?>
        </h2>

    </div>


    <div class="card">

        <h3>Total Pendapatan</h3>

        <h2>
            Rp
            <?php
            echo number_format(
                $income['income'] ?? 0,
                0,
                ',',
                '.'
            );
            ?>
        </h2>

    </div>

</div>

<?php endif; ?>


<!-- =========================================================
     PRODUK
========================================================= -->

<?php if($page == 'products'): ?>

<div class="topbar">

    <h1>📦 Manajemen Produk</h1>

</div>

<div class="flex">

<div class="w-50">

<form method="POST">

    <input
        type="text"
        name="name"
        placeholder="Nama Produk"
        required
    >

    <input
        type="number"
        name="price"
        placeholder="Harga"
        min="0"
        required
    >

    <input
        type="number"
        name="stock"
        placeholder="Stock"
        min="0"
        required
    >

    <input
        type="text"
        name="category"
        placeholder="Kategori"
        required
    >

    <input
        type="date"
        name="expired_date"
        required
    >

    <button name="add_product">
        + Tambah Produk
    </button>

</form>

</div>


<div class="w-50">

<table>

<tr>
    <th>ID</th>
    <th>Nama</th>
    <th>Harga</th>
    <th>Stock</th>
    <th>Aksi</th>
</tr>

<?php

$products = mysqli_query(
    $conn,
    "SELECT * FROM products ORDER BY id DESC"
);

while($p = mysqli_fetch_assoc($products)):

?>

<tr>

    <td>
        <?php echo $p['id']; ?>
    </td>

    <td>
        <?php echo htmlspecialchars($p['name']); ?>
    </td>

    <td>
        Rp
        <?php
        echo number_format(
            $p['price'],
            0,
            ',',
            '.'
        );
        ?>
    </td>

    <td>
        <?php echo $p['stock']; ?>
    </td>

    <td>

        <a
            href="index.php?delete_product=<?php echo $p['id']; ?>"
            onclick="return confirm('Hapus produk ini?')"
        >

            <button>
                Hapus
            </button>

        </a>

    </td>

</tr>

<?php endwhile; ?>

</table>

</div>

</div>

<?php endif; ?>


<!-- =========================================================
     CUSTOMER
========================================================= -->

<?php if($page == 'customers'): ?>

<div class="topbar">

    <h1>👥 Customer</h1>

</div>

<div class="flex">

<div class="w-50">

<form method="POST">

    <input
        type="text"
        name="name"
        placeholder="Nama Customer"
        required
    >

    <input
        type="text"
        name="phone"
        placeholder="Nomor HP"
        required
    >

    <textarea
        name="address"
        placeholder="Alamat"
    ></textarea>

    <input
        type="email"
        name="email"
        placeholder="Email"
    >

    <button name="add_customer">
        + Tambah Customer
    </button>

</form>

</div>


<div class="w-50">

<table>

<tr>
    <th>ID</th>
    <th>Nama</th>
    <th>HP</th>
    <th>Aksi</th>
</tr>

<?php

$customers = mysqli_query(
    $conn,
    "SELECT * FROM customers ORDER BY id DESC"
);

while($c = mysqli_fetch_assoc($customers)):

?>

<tr>

    <td>
        <?php echo $c['id']; ?>
    </td>

    <td>
        <?php echo htmlspecialchars($c['name']); ?>
    </td>

    <td>
        <?php echo htmlspecialchars($c['phone']); ?>
    </td>

    <td>

        <a
            href="index.php?delete_customer=<?php echo $c['id']; ?>"
            onclick="return confirm('Hapus customer ini?')"
        >

            <button>
                Hapus
            </button>

        </a>

    </td>

</tr>

<?php endwhile; ?>

</table>

</div>

</div>

<?php endif; ?>


<!-- =========================================================
     KASIR
========================================================= -->

<?php if($page == 'cashier'): ?>

<div class="topbar">

    <h1>🛒 Kasir</h1>

</div>


<?php if(isset($error_checkout)): ?>

<div class="danger">

    <?php echo $error_checkout; ?>

</div>

<?php endif; ?>


<div class="grid-product">

<?php

$products = mysqli_query(
    $conn,
    "SELECT * FROM products
     WHERE stock > 0
     ORDER BY id DESC"
);

while($p = mysqli_fetch_assoc($products)):

?>

<div class="product-card">

    <h3>
        <?php echo htmlspecialchars($p['name']); ?>
    </h3>

    <p style="margin-top:10px;">
        Harga :
        <b>
            Rp
            <?php
            echo number_format(
                $p['price'],
                0,
                ',',
                '.'
            );
            ?>
        </b>
    </p>

    <p style="margin-top:8px;">
        Stock :
        <?php echo $p['stock']; ?>
    </p>

    <a
        href="index.php?page=cashier&add_cart=<?php echo $p['id']; ?>"
    >

        <button>
            + Tambah
        </button>

    </a>

</div>

<?php endwhile; ?>

</div>


<h2 style="margin-top:40px;">
    🛍️ Keranjang
</h2>


<table>

<tr>

    <th>No</th>
    <th>Nama</th>
    <th>Harga</th>
    <th>Qty</th>
    <th>Subtotal</th>
    <th>Aksi</th>

</tr>


<?php

$no = 1;
$total = 0;

foreach($_SESSION['cart'] as $index => $cart):

$subtotal = $cart['price'] * $cart['qty'];

$total += $subtotal;

?>

<tr>

    <td>
        <?php echo $no++; ?>
    </td>

    <td>
        <?php echo htmlspecialchars($cart['name']); ?>
    </td>

    <td>
        Rp
        <?php
        echo number_format(
            $cart['price'],
            0,
            ',',
            '.'
        );
        ?>
    </td>

    <td>
        <?php echo $cart['qty']; ?>
    </td>

    <td>
        Rp
        <?php
        echo number_format(
            $subtotal,
            0,
            ',',
            '.'
        );
        ?>
    </td>

    <td>

        <a
            href="index.php?page=cashier&remove_cart=<?php echo $index; ?>"
        >

            <button>
                Hapus
            </button>

        </a>

    </td>

</tr>

<?php endforeach; ?>


<tr>

    <td colspan="4">
        <b>Total</b>
    </td>

    <td colspan="2">

        <b>
            Rp
            <?php
            echo number_format(
                $total,
                0,
                ',',
                '.'
            );
            ?>
        </b>

    </td>

</tr>

</table>


<form
    method="POST"
    style="margin-top:20px;"
>

    <input
        type="text"
        name="customer_name"
        placeholder="Nama Customer"
        required
    >

    <input
        type="number"
        name="pay"
        placeholder="Jumlah Bayar"
        min="<?php echo $total; ?>"
        required
    >

    <button name="checkout">

        💳 Checkout

    </button>

</form>

<?php endif; ?>


<!-- =========================================================
     TRANSAKSI
========================================================= -->

<?php if($page == 'transactions'): ?>

<div class="topbar">

    <h1>🧾 Riwayat Transaksi</h1>

</div>


<table>

<tr>

    <th>No</th>
    <th>Invoice</th>
    <th>Customer</th>
    <th>Total</th>
    <th>Tanggal</th>

</tr>


<?php

$no = 1;

$transactions = mysqli_query(
    $conn,
    "SELECT * FROM transactions ORDER BY id DESC"
);

while($t = mysqli_fetch_assoc($transactions)):

?>

<tr>

    <td>
        <?php echo $no++; ?>
    </td>

    <td>
        <?php echo htmlspecialchars($t['invoice']); ?>
    </td>

    <td>
        <?php echo htmlspecialchars($t['customer_name']); ?>
    </td>

    <td>
        Rp
        <?php
        echo number_format(
            $t['total'],
            0,
            ',',
            '.'
        );
        ?>
    </td>

    <td>
        <?php echo $t['created_at']; ?>
    </td>

</tr>

<?php endwhile; ?>

</table>

<?php endif; ?>


<!-- =========================================================
     STRUK
========================================================= -->

<?php if($page == 'receipt'): ?>

<?php

if(isset($_SESSION['last_invoice'])) {

    $invoice = mysqli_real_escape_string(
        $conn,
        $_SESSION['last_invoice']
    );

    $data = mysqli_fetch_assoc(
        mysqli_query(
            $conn,
            "SELECT * FROM transactions
             WHERE invoice='$invoice'"
        )
    );

    $details = mysqli_query(
        $conn,
        "SELECT * FROM transaction_details
         WHERE invoice='$invoice'"
    );

?>

<div class="receipt">

    <h2>🥗 SALAD SATURDAY</h2>

    <p style="text-align:center;">
        STRUK BELANJA
    </p>

    <br>

    <p>
        Invoice :
        <?php echo $data['invoice']; ?>
    </p>

    <p>
        Customer :
        <?php echo htmlspecialchars($data['customer_name']); ?>
    </p>

    <p>
        Tanggal :
        <?php echo $data['created_at']; ?>
    </p>

    <hr>

    <table>

    <tr>

        <th>Produk</th>
        <th>Qty</th>
        <th>Total</th>

    </tr>

    <?php while($d = mysqli_fetch_assoc($details)): ?>

    <tr>

        <td>
            <?php
            echo htmlspecialchars(
                $d['product_name']
            );
            ?>
        </td>

        <td>
            <?php echo $d['qty']; ?>
        </td>

        <td>
            Rp
            <?php
            echo number_format(
                $d['subtotal'],
                0,
                ',',
                '.'
            );
            ?>
        </td>

    </tr>

    <?php endwhile; ?>

    </table>

    <hr>

    <h3>
        Total :
        Rp
        <?php
        echo number_format(
            $data['total'],
            0,
            ',',
            '.'
        );
        ?>
    </h3>

    <h3>
        Bayar :
        Rp
        <?php
        echo number_format(
            $data['pay'],
            0,
            ',',
            '.'
        );
        ?>
    </h3>

    <h3>
        Kembalian :
        Rp
        <?php
        echo number_format(
            $data['change_money'],
            0,
            ',',
            '.'
        );
        ?>
    </h3>

    <br>

    <button onclick="window.print()">
        🖨️ Print
    </button>

    <a href="index.php?page=transactions">

        <button>
            Kembali
        </button>

    </a>

</div>

<?php

}

endif;

?>

</div>

</body>
</html>
