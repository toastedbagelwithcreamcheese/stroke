BACKEND 1
const express = require('express');
const mysql = require('mysql');
const path = require('path'); 
const app = express();
app.use(express.static("public"));
const port = 3000

//mysql adatbázis-kapcsolat
const db = mysql.createConnection({
    host: 'localhost',
    user: 'root',
    password: '',
    database: 'ackiado2'
});

//elindul a szerver
app.listen(port,() => {
    console.log(`A szerver elindult a ${port}-es porton...`);  
});

app.get('/get_html/:filename', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', req.params.filename));
});

// Összes tábla nevének lekérdezése
app.get('/get_tablak', (req, res) => {
	const sql = "SHOW TABLES";
    kuld(res, lekerdez(sql));
});

// Egy adott tábla adatainak lekérdezése
app.get('/get_tablak/:tabla', (req, res) => {
	const tabla = req.params.tabla;
    const sql = `SELECT * FROM ${tabla}`;
    kuld(res, lekerdez(sql));
});

app.get('/bevetel', (req, res) => {
	const id = req.query.id;
	const sql = `SELECT 
                    COALESCE(SUM(re.darab), 0) AS OsszesDarab, 
                    COALESCE(SUM(re.darab * r.ar), 0) AS OsszesBevetel 
                 FROM regeny r 
                 LEFT JOIN rendeles re ON r.id = re.regenyid 
                 WHERE r.id = ${id}`;
	kuld(res, lekerdez(sql));
});

app.get('/foszereplo_csop', (req, res) => {
	const sql = `SELECT 
					CASE 
						WHEN kategoria IS NULL OR kategoria = '' THEN '(nem ismert)'
						ELSE kategoria
					END AS Főszereplő,
					COUNT(*) AS "Regények száma"
					FROM regeny
					GROUP BY 
					CASE 
						WHEN kategoria IS NULL OR kategoria = '' THEN '(nem ismert)'
						ELSE kategoria
					END;`;
	kuld(res, lekerdez(sql));
});

app.get('/magyarcimek', (req, res) => {
	const sql = 'SELECT DISTINCT * FROM regeny ORDER BY regeny.magyar;;'
	kuld(res, lekerdez(sql));
});

app.get('/get_table_name/:id', (req, res) => {
	const id = req.params.id;
	const sql = `SELECT rendeles.datum, rendeles.darab
					FROM regeny, rendeles 
					WHERE regeny.id = rendeles.regenyid
					AND rendeles.darab > 1
					AND regeny.id = ${id}
					ORDER BY rendeles.datum`;
	kuld(res, lekerdez(sql));
});

function lekerdez(sql) {
	return new Promise((resolve, reject) => {
		db.query(sql, (err, result) => {
			if (err) reject(err);
			else resolve(result);
		});
	});
}

function kuld(res, promise) {
	promise.then((result) => {
		res.json(result);
	}).catch((err) => {
		res.json(err);
	});
}

BACKEND 2
const express = require('express');
const mysql = require('mysql');
const path = require('path');
const app = express();
app.use(express.static('public'));
const port = 9041;

const db = mysql.createConnection({
	host: 'localhost',
	user: 'root',
	password: '',
	database: 'vetitesek'
});

app.listen(port, () => {
	console.log(`Server is running at http://localhost:${port}`);
});

function lekerdez(sql) {
	return new Promise((resolve, reject) => {
		db.query(sql, (err, result) => {
			if (err) reject(err);
			else resolve(result);
		});
	});
}

function kuld(res, promise) {
	promise.then((result) => {
		res.json(result);
	}).catch((err) => {
		res.json(err);
	});
}

app.get('/get_html/:filename', (req, res) => {
	res.sendFile(path.join(__dirname, 'public', req.params.filename));
});

app.get('/get_tablak', (req, res) => {
	const sql = 'SHOW TABLES';
	kuld(res, lekerdez(sql));
});

app.get('/get_tablak/:tabla', (req, res) => {
	const sql = `SELECT * FROM ${req.params.tabla}`;
	kuld(res, lekerdez(sql));
});

app.get('/get_moziid_count', (req, res) => {
	const sql = `SELECT moziid, COUNT(moziid) count FROM eloadas GROUP BY moziid`;
	kuld(res, lekerdez(sql));
});

app.get('/get_eloadasok/:id', (req, res) => {
	const sql = `SELECT kezdes Kezdés, szinkron Szinkron FROM eloadas WHERE moziid = ${req.params.id} ORDER BY kezdes`;
	kuld(res, lekerdez(sql));
});

app.get('/get_szinkron_tipusonkent', (req, res) => {
	const sql = `SELECT szinkron Szinkron, COUNT(szinkron) "Vetítések száma" FROM eloadas GROUP BY szinkron`;
	kuld(res, lekerdez(sql));
});

FRONTEND 1
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
	<script src="index.js"></script>
	<link rel="stylesheet" href="https://cdn.datatables.net/2.2.2/css/dataTables.dataTables.css" />
	<script src="https://cdn.datatables.net/2.2.2/js/dataTables.js"></script>
	<link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css">
  <title>A. C. Kiadó</title>
  <style>
    nav a {
      cursor: pointer;
    }
  </style>
</head>
<body>
  <nav class="navbar navbar-expand navbar-light bg-light mb-2">
		<div class="nav navbar-nav">
			<a class="nav-item nav-link" onclick="load('tablak.html')">Táblák</a>
			<a class="nav-item nav-link" onclick="load('tipusonkent.html')">Típusonként</a>
			<a class="nav-item nav-link" onclick="load('regenyek.html')">Regények és rendelések</a>
		</div>
	</nav>
  <div id="content" ures="true"></div>
</body>
</html>
<div class="row col-12" style="height: calc(100vh - 100px);">
    <div class="col-auto h-100">
        <h2 class="">!</h2>
        <select id="selectCim"></select>
    </div>
    <div class="col-auto h-100" style="overflow: auto;">
        <h2 class=""></h2>
        <table class="table table-striped table-bordered table-hover table-responsive-sm">
            <thead id="tablazatHead"></thead>
            <tbody id="tablazatBody"></tbody>
        </table>
    </div>
</div>
<script>
    var xhrVaros = new XMLHttpRequest();
    xhrVaros.open('GET', '/magyarcimek', true);
    xhrVaros.send();
    xhrVaros.onreadystatechange = function() {
        if (xhrVaros.readyState == 4 && xhrVaros.status == 200) {
            
            var cimek = JSON.parse(xhrVaros.responseText);
            console.log(cimek);
            var opciok = '<option></option>';
            cimek.forEach(d => {
                opciok += `<option value="${d.id}">${d.magyar}</option>`;
            });
            $('#selectCim').html(opciok);
        }
    };
    $('#selectCim').change(function() {
        const valasztott = $('#selectCim').val();
        fillTable('tablazatHead', 'tablazatBody', '/get_table_name/' + valasztott);
    });
</script>
<div class="row col-12 justify-content-center" style="height: calc(100vh - 100px);">
    <div class="col-lg-5 h-100 mb-5" style="overflow: auto;">
        <h2 id="tablazatCim0" class="tablazatCim"></h2>
        <table id="tablazat0" class="table table-striped table-bordered table-hover table-responsive-xs">
            <thead id="tablazatHead0"></thead>
            <tbody id="tablazatBody0"></tbody>
        </table>
    </div>
    <div class="col-lg-5 h-100 mb-5" style="overflow: auto;">
        <h2 id="tablazatCim1" class="tablazatCim"></h2>
        <table id="tablazat1" class="table table-striped table-bordered table-hover table-responsive-xs">
            <thead id="tablazatHead1"></thead>
            <tbody id="tablazatBody1"></tbody>
        </table>
    </div>
</div>
<script>
    var xhrTablak = new XMLHttpRequest();
    xhrTablak.open('GET', '/get_tablak', true);
    xhrTablak.send();
    xhrTablak.onreadystatechange = function() {
        if (xhrTablak.readyState == 4 && xhrTablak.status == 200) {
            var tablak = JSON.parse(xhrTablak.responseText);
            tablaRekurziv(0, tablak);
        }
    };

    function tablaRekurziv(i, tablak) {
        if (i < tablak.length) {
           $("#tablazatCim" + i).text(tablak[i].Tables_in_ackiado2);
            fillTable('tablazatHead' + i, 'tablazatBody' + i, 'get_tablak/' + tablak[i].Tables_in_ackiado2).then((result) => {
                tablaRekurziv(i + 1, tablak);
            });
        }
    } 
</script>
<div class="row col-12 justify-content-center" style="height: calc(100vh - 100px);">
    <div class="col-auto h-100" style="overflow: auto;">
        <table class="table table-striped table-bordered table-hover table-responsive-sm">
            <thead id="tablazatHead"></thead>
            <tbody id="tablazatBody"></tbody>
        </table>
    </div>
</div>
<script>
    fillTable('tablazatHead', 'tablazatBody', 'foszereplo_csop');
</script>

FRONTEND JS 1
$(document).ready(function() {
    if ($("#content").attr("ures") == "true") {
        load();
        $("#content").attr("ures", "false");
    }
});

function load(url) {
    const urlToLoad = url ? url : 'tablak.html';
    const xhr = new XMLHttpRequest();
    xhr.open('GET', '/get_html/' + urlToLoad, true);
    xhr.send();
    xhr.onreadystatechange = function() {
        if (xhr.readyState == 4 && xhr.status == 200) {
            $("#content").html(xhr.responseText);
        }
    };
}

function fillTable(tablehead, tablebody, url) {
    return fetch(url)
        .then(res => res.json())
        .then(data => {
            if (!Array.isArray(data) || data.length === 0) return;

            let fejlec = '<tr>';
            for (let key in data[0]) {
                fejlec += `<th>${key}</th>`;
            }
            fejlec += '</tr>';

            let sorok = '';
            data.forEach(sor => {
                sorok += '<tr>';
                for (let key in sor) {
                    sorok += `<td>${sor[key]}</td>`;
                }
                sorok += '</tr>';
            });

            $(`#${tablehead}`).html(fejlec);
            $(`#${tablebody}`).html(sorok);
        })
        .catch(error => {
            console.error('Hiba a tábla betöltése közben:', error);
        });
}

function tooltipcucc(event) {
    const sor = event.target.closest('tr');
    if (!sor) return;
    const tabla = sor.closest('table');
    const tablaNev = tabla.id.replace('Tabla', '');
    console.log(tablaNev);
    const mezo = Array.from(sor.children).map((cella) => cella.innerText);
    console.log(mezo);
    if (tablaNev === "tablazat0"){
        const id = mezo[0];
        $.get(`/bevetel?id=${id}`, function(data) {
             if (data && data.length > 0) {
            const osszesDarab = data[0].OsszesDarab;
            const osszesBevetel = data[0].OsszesBevetel;
            $(sor).attr(
                "title",
                `Összes darab: ${osszesDarab}, Összes bevétel: ${osszesBevetel} Ft`
            );
        }
        })
    }
}
$(document).on('mouseover', '#tablazat0 tbody tr', tooltipcucc);

FRONTEND 2
<!DOCTYPE html>
<html lang="hu">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
	<script src="index.js"></script>
	<link rel="stylesheet" href="https://cdn.datatables.net/2.2.2/css/dataTables.dataTables.css" />
	<script src="https://cdn.datatables.net/2.2.2/js/dataTables.js"></script>
	<link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css">
	<title>Vetítések</title>
	<style>
		nav a {
			cursor: pointer;
		}
	</style>
</head>
<body>
	<nav class="navbar navbar-expand navbar-light bg-light mb-2">
		<div class="nav navbar-nav">
			<a class="nav-item nav-link" onclick="load('tablak.html')">Táblák</a>
			<a class="nav-item nav-link" onclick="load('vetitesek.html')">Mozik és vetítések</a>
			<a class="nav-item nav-link" onclick="load('szinkron.html')">Szinkron típusonként</a>
		</div>
	</nav>
	<div id="content" ures=true>
	</div>

</body>
</html>
<div class="row col-12 justify-content-center" style="height: calc(100vh - 100px);">
	<div class="col-auto h-100" style="overflow: auto;">
		<table class="table table-striped table-bordered table-hover table-responsive-sm">
			<thead id="tablazatHead"></thead>
			<tbody id="tablazatBody"></tbody>
		</table>
	</div>
</div>
<script>
	fillTable('tablazatHead', 'tablazatBody', 'get_szinkron_tipusonkent');
</script>
<div class="row col-12 justify-content-center" style="height: calc(100vh - 100px);">
	<div class="col-lg-5 h-100 mb-5" style="overflow: auto;">
		<h2 id="tablazatCim0" class="tablazatCim"></h2>
		<table id="tablazat0" class="table table-striped table-bordered table-hover table-responsive-xs">
			<thead id="tablazatHead0"></thead>
			<tbody id="tablazatBody0"></tbody>
		</table>
	</div>
	<div class="col-lg-5 h-100 mb-5" style="overflow: auto;">
		<h2 id="tablazatCim1" class="tablazatCim"></h2>
		<table id="tablazat1" class="table table-striped table-bordered table-hover table-responsive-xs">
			<thead id="tablazatHead1"></thead>
			<tbody id="tablazatBody1"></tbody>
		</table>
	</div>
</div>
<script>
	var xhrTablak = new XMLHttpRequest();
	xhrTablak.open('GET', '/get_tablak', true);
	xhrTablak.send();
	xhrTablak.onreadystatechange = function() {
		if (xhrTablak.readyState == 4 && xhrTablak.status == 200) {
			var tablak = JSON.parse(xhrTablak.responseText);
			tablaRekurziv(0, tablak);
		}
	};

	function tablaRekurziv(i, tablak) {
		if (i < tablak.length) {
			$("#tablazatCim" + i).text(tablak[i].Tables_in_vetitesek + " tábla");
			fillTable('tablazatHead' + i, 'tablazatBody' + i, 'get_tablak/' + tablak[i].Tables_in_vetitesek).then((result) => {
				tablaRekurziv(i + 1, tablak);
			});
		}
	} 
</script>
<div class="row col-12" style="height: calc(100vh - 100px);">
	<div class="col-auto h-100">
		<h2 class="">Válasszon mozit!</h2>
		<select id="selectMozi"></select>
	</div>
	<div class="col-auto h-100" style="overflow: auto;">
		<h2 class="">Ezekre az előadásokra van szabad hely:</h2>
		<table class="table table-striped table-bordered table-hover table-responsive-sm">
			<thead id="tablazatHead"></thead>
			<tbody id="tablazatBody"></tbody>
		</table>
	</div>
</div>
<script>
	var xhrVaros = new XMLHttpRequest();
	xhrVaros.open('GET', '/get_tablak/mozi', true);
	xhrVaros.send();
	xhrVaros.onreadystatechange = function() {
		if (xhrVaros.readyState == 4 && xhrVaros.status == 200) {
			var mozik = JSON.parse(xhrVaros.responseText);
			var opciok = '<option>Válasszon</option>';
			mozik.forEach(d => {
				opciok += `<option value="${d.id}">${d.nev}; ${d.irszam} ${d.varos} ${d.cim}</option>`;
			});
			$('#selectMozi').html(opciok);
		}
	};
	$('#selectMozi').change(function() {
		const moziid = $('#selectMozi').val();
		console.log(moziid);
		fillTable('tablazatHead', 'tablazatBody', '/get_eloadasok/' + moziid);
	});
</script>

FRONTEND JS 2
$(document).ready(function() {
	if ($("#content").attr("ures") == "true") {
		load();
		$("#content").attr("ures", "false");
	}
});

function load(url) {
	const urlToLoad = url ? url : 'tablak.html';
	const xhr = new XMLHttpRequest();
	xhr.open('GET', '/get_html/' + urlToLoad, true);
	xhr.send();
	xhr.onreadystatechange = function() {
		if (xhr.readyState == 4 && xhr.status == 200) {
			$("#content").html(xhr.responseText);
		}
	};
}

function selectTable() {
	$('#tablazatHead #tablazatBody #tablazatHead2 #tablazatBody2').html('');
	fillTable('tablazatHead', 'tablazatBody', '/get_tablak/' + $('#lista').val(), 'onclick="sortElemek(event)"', 'onclick="sorKatt(event)"');
}

function sortElemek(event, tablaNev) {
	if (tablaNev == 'jarat') {
		const column = $(event.target).index();
		const table = $('#tablazatBody1');
		const rows = table.find('tr').get();
		rows.sort(function(a, b) {
			const A = $(a).children('td').eq(column).text().toUpperCase();
			const B = $(b).children('td').eq(column).text().toUpperCase();
			if (A < B) return -1;
			if (A > B) return 1;
			return 0;
		});

		$.each(rows, function(index, row) { table.append(row); });
	}
}

function sorKatt(event) {
	const tableName = $('#lista').val();
	if (tableName === 'kerulet') {
		const row = $(event.target).parent().children('td').get();
		const id = $(row[0]).text();
		fillTable('tablazatHead2', 'tablazatBody2', `/get_varosreszek/${id}`);
	}
}

function masikTabla() {
	$('#tablazatHead #tablazatBody #tablazatHead2 #tablazatBody2').html('');
	fillTable('tablazatHead', 'tablazatBody', '/get_masik_tabla');
}

function fillTable(tablehead, tablebody, url) {
	return new Promise((resolve, reject) => {
		$(`#${tablehead}`).html('');
		$(`#${tablebody}`).html('');
		xhr = new XMLHttpRequest();
		xhr.open('GET', url, true);
		xhr.send();
		xhr.onreadystatechange = function() {
			var tabla = JSON.parse(xhr.responseText);
			if (xhr.readyState == 4 && xhr.status == 200) {
				if (url.search("/mozi") != -1) {
					xhr = new XMLHttpRequest();
					xhr.open('GET', '/get_moziid_count', true);
					xhr.send();
					xhr.onreadystatechange = function() {
						if (xhr.readyState == 4 && xhr.status == 200) {
							var moziidcount = JSON.parse(xhr.responseText);
							var fejlec = '<tr>';
							for (var key in tabla[0]) {
								fejlec += `<th>` + key + '</th>';
							}
							fejlec += '</tr>';
							var sorok = '';
							for (var i = 0; i < tabla.length; i++) {
								let szam = moziidcount.filter(m => m.moziid == tabla[i].id);
								sorok += `<tr title="előadások száma: ${szam.length > 0 ? szam[0].count : 0}">`
								for (var k in tabla[i]) {
									sorok += '<td>' + tabla[i][k] + '</td>';
								}
								sorok += '</tr>';
								
							}
							$(`#${tablehead}`).html(fejlec);
							$(`#${tablebody}`).html(sorok);
							resolve();
						}
					}
				} else {
					var fejlec = '<tr>';
					for (var key in tabla[0]) {
						fejlec += `<th>` + key + '</th>';
					}
					fejlec += '</tr>';
					var sorok = '';
					for (var i = 0; i < tabla.length; i++) {
						sorok += `<tr>`;
						for (var k in tabla[i]) {
							sorok += '<td>' + tabla[i][k] + '</td>';
						}
						sorok += '</tr>';
					}
					$(`#${tablehead}`).html(fejlec);
					$(`#${tablebody}`).html(sorok);
					resolve();
				}
			}
		};
	});
}
