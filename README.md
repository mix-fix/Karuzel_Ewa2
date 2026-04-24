<?php
session_start();

// категории
$categories = [
    "DN LL", "DW RL", "GM RL", "GM LL",
    "GN RR", "GN LL", "GX LL", "GX RR",
    "TM LL", "NN RL", 
];

// инициализация
if (!isset($_SESSION['stan'])) {
    foreach ($categories as $cat) {
        $_SESSION['stan'][$cat] = 0;
        $_SESSION['total'][$cat] = 0;
    }
}

// линия
if (!isset($_SESSION['line'])) {
    $_SESSION['line'] = array_fill(0, 10, [
        'color' => 'gray',
        'type' => null,
        'text' => ''
    ]);
}

// обработка POST
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
	// ➤ СОХРАНЕНИЕ НОРМЫ
if (isset($_POST['norma'])) {
    $_SESSION['norma'] = (int)$_POST['norma'];
}

    

    $input = trim($_POST['input']);

    // ✅ СТРОГАЯ ПРОВЕРКА ФОРМАТА
    if (
    isset($_POST['input']) &&
    $_POST['input'] !== '' &&
    !isset($_POST['clear_line']) &&
    !isset($_POST['reset'])
) {

    

    // ✅ проверка
    if (preg_match('/^(DN|DW|GM|GN|GX|TM|NN)\s+(LL|RL|RR)\s+\d{4}\s+(зел(ений|ёный)|жел(тий|тый))$/iu', $input)) {

        // 👉 ТОЛЬКО ЕСЛИ ПРОШЛО — выполняем код

        // убираем "жел/зел"
        $display = preg_replace('/\s*(жел.*|зел.*)/iu', '', $input);

        $parts = explode(" ", $input);
        $key = null;

        if (count($parts) >= 2) {
            $key = $parts[0] . " " . $parts[1];

            if (isset($_SESSION['total'][$key])) {
                $_SESSION['total'][$key]++;
            }
        }

        // цвет
        $color = "gray";
        if (stripos($input, "жел") !== false) $color = "yellow";
        if (stripos($input, "зел") !== false) $color = "green";

        // сдвиг линии
        array_shift($_SESSION['line']);
        $_SESSION['line'][] = [
            'color' => $color,
            'type' => $key,
            'text' => $display
        ];
    }

    // ❌ если не прошло — просто ничего не делаем
}

    // ➤ СБРОС "ВСЕГО"
    if (isset($_POST['reset'])) {
        foreach ($categories as $cat) {
            $_SESSION['total'][$cat] = 0;
        }
    }

    // ➤ ОЧИСТКА СХЕМЫ
    if (isset($_POST['clear_line'])) {
        $_SESSION['line'] = array_fill(0, 10, [
            'color' => 'gray',
            'type' => null,
            'text' => ''
        ]);
    }
}

// ➤ ПЕРЕСЧЁТ Stan (уникальные)
foreach ($categories as $cat) {
    $_SESSION['stan'][$cat] = 0;
}

$unique = [];

foreach ($_SESSION['line'] as $item) {
    if (!empty($item['type'])) {
        $unique[$item['type']] = true;
    }
}

foreach ($unique as $type => $_) {
    if (isset($_SESSION['stan'][$type])) {
        $_SESSION['stan'][$type] = 1;
    }
}

// ➤ ОБЩЕЕ КОЛ-ВО
$total_all = array_sum($_SESSION['total']);

// ➤ НОРМА
$norma = $_SESSION['norma'] ?? 0;

// ➤ ОСТАЛОСЬ
$left = max(0, $norma - $total_all);
?>

<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Мониторинг</title>

<style>
body {
    font-family: Arial;
    background: #1e1e1e;
    color: #fff;
    margin: 0;
}

.container {
    width: 95%;
    margin: 20px auto;
    text-align: center;
}

.input-box {
    display: flex;
    justify-content: center;
}

.controls input {
    width: 120px;
    padding: 6px;
    font-size: 12px;
}

.controls button {
    padding: 4px 8px;
    font-size: 12px;
}

.box {
    border: 3px solid #fff;
    padding: 20px;
    background: #2b2b2b;
    height: 60vh;
    overflow-y: auto;
}

table {
    width: 100%;
    font-size: 27px;
}

td, th {
    text-align: left;
    padding: 10px;
}

td:nth-child(2) { color: #4caf50; }
td:nth-child(3) { color: #ffc107; }

/* линия */
.line {
    margin-top: 20px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    width: 100%;
}

.row {
    display: flex;
    gap: 10px;
    width: 100%;
}

.block {
    flex: 1;
    height: 50px;      /* было 80 → меньше */
    border: 2px solid #fff;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 14px;   /* уменьшили текст */
    font-weight: bold;
    text-align: center;
    color: #000;
}

.yellow { background: yellow; }
.green { background: green; }
.gray { background: #444; }

#clock {
    font-size: 80px;      /* размер (можешь 70–80 если нужно ещё больше) */
    font-weight: bold;
    text-align: center;   /* по центру */
    margin: 50px 20;
    letter-spacing: 3px;  /* немного растянуть цифры */
}

#shift {
    position: absolute;
    top: 10px;
    left: 20px;
    font-size: 40px;
    font-weight: bold;
}


/* верхний правый блок */
.controls {
    position: fixed;
    top: 10px;
    right: 80px;
    display: flex;
    gap: 5px;
}

/* кнопка ⚙️ */
#togglePanel {
    position: fixed;
    top: 10px;
    right: 20px;
    z-index: 1000;
}

/* панель */
#sidePanel {
    position: fixed;
    top: 50px;
    right: -220px;
    width: 200px;
    background: #2b2b2b;
    border: 2px solid #fff;
    padding: 10px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    transition: right 0.3s;
}

/* открыта */
#sidePanel.active {
    right: 20px;
}

.stats {
    margin: 15px 0;
    font-size: 32px;
    display: flex;
    justify-content: space-around;
    font-weight: bold;
}

#normaBlock {
    font-size: 35px;
    font-weight: bold;
    text-align: center;
    margin-top: -10px;
}
</style>

</head>
<body>
<div id="clock"></div>
<div id="normaBlock">
    Norma: <?php echo $norma; ?>
</div>
<div id="shift"></div>

<div class="container">

<form method="POST">

    <!-- верхний правый блок -->
    <div class="controls">
    <input type="text" name="input" placeholder="GN RR 8002 green" autofocus>
    <input type="number" name="norma" placeholder="Норма"
    value="<?php echo $_SESSION['norma'] ?? ''; ?>">
</div>

    <!-- кнопка ⚙️ -->
    <button type="button" id="togglePanel">⚙️</button>

    <!-- выезжающая панель -->
    <div id="sidePanel">
        <button type="submit" name="reset">Resetuj</button>
        <button type="submit" name="clear_line">Resetuj schemat</button>
		
    </div>

</form>

<div class="box">
    <table>
        <tr>
            <th>Deska</th>
            <th>Bieżące</th>
            <th>Łącznie</th>
        </tr>

        <?php foreach ($categories as $cat): ?>
        <tr>
            <td><?php echo $cat; ?></td>
            <td><?php echo $_SESSION['stan'][$cat]; ?> szt</td>
            <td><?php echo $_SESSION['total'][$cat]; ?> szt</td>
        </tr>
        <?php endforeach; ?>
    </table>
</div>
<div class="stats">
    <div>ZROBIONO W TEJ ZMIANIE: <?php echo $total_all; ?></div>
    <div>POZOSTAŁO: <?php echo $left; ?></div>
    
</div>
<!-- линия -->
<div class="line">

    <!-- верх -->
    <div class="row">
        <?php for ($i = 4; $i >= 0; $i--): ?>
            <div class="block <?php echo $_SESSION['line'][$i]['color']; ?>">
                <?php echo $_SESSION['line'][$i]['text']; ?>
            </div>
        <?php endfor; ?>
    </div>

    <!-- низ -->
    <div class="row">
        <?php for ($i = 5; $i < 10; $i++): ?>
            <div class="block <?php echo $_SESSION['line'][$i]['color']; ?>">
                <?php echo $_SESSION['line'][$i]['text']; ?>
            </div>
        <?php endfor; ?>
    </div>

</div>
<script>
function updateClock() {
    const now = new Date();

    const h = String(now.getHours()).padStart(2, '0');
    const m = String(now.getMinutes()).padStart(2, '0');
    const s = String(now.getSeconds()).padStart(2, '0');

    document.getElementById('clock').textContent = `${h}:${m}:${s}`;
}

setInterval(updateClock, 1000);
updateClock();
</script>
<script>
function updateShift() {
    const now = new Date();
    const h = now.getHours();
    const m = now.getMinutes();
    const time = h * 60 + m; // минуты от начала суток

    let shift = "";

    // 1 смена: 06:30 - 14:20
    if (time >= (6*60+30) && time <= (14*60+20)) {
        shift = "Zmiana 1🌅 ";
    }
    // 2 смена: 14:30 - 22:20
    else if (time >= (14*60+30) && time <= (22*60+20)) {
        shift = "Zmiana 2☀️ ";
    }
    // 3 смена: 22:30 - 06:20 (через ночь)
    else {
        shift = "Zmiana 3🌙 ";
    }

    document.getElementById("shift").textContent = shift;
}

// обновление каждую секунду
setInterval(updateShift, 1000);
updateShift();
</script>
<script>
let focusTimer;

function setAutoFocus() {
    clearTimeout(focusTimer);

    focusTimer = setTimeout(function () {
        const input = document.querySelector('input[name="input"]');
        if (input && document.activeElement !== input) {
            input.focus();
        }
    }, 2000); // задержка 2 секунды
}

// при загрузке
document.addEventListener("DOMContentLoaded", function () {
    setAutoFocus();
});

// при любом клике на странице
document.addEventListener("click", function () {
    setAutoFocus();
});

// при потере фокуса
document.addEventListener("focusout", function () {
    setAutoFocus();
});
</script>
<script>
const btn = document.getElementById("togglePanel");
const panel = document.getElementById("sidePanel");

btn.addEventListener("click", function(e) {
    panel.classList.toggle("active");
});
</script>
</body>
<script>
let timer;

const input = document.querySelector('input[name="input"]');

input.addEventListener("input", function () {
    clearTimeout(timer);

    timer = setTimeout(() => {
        if (input.value.trim() !== "") {
            input.form.submit();
        }
    }, 800); // задержка (можно 500–1500)
});
</script>
</html>
