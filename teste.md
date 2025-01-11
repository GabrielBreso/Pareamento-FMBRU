<!DOCTYPE html>
<html lang="pt-br">

<head>
    <meta charset="UTF-8">
    <title>Comparação de Respostas</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
            color: #333;
        }

        header {
            background-color: #4CAF50;
            color: white;
            padding: 10px 0;
            text-align: center;
        }

        main {
            padding: 20px;
            max-width: 800px;
            margin: auto;
        }

        h1 {
            text-align: center;
        }

        form {
            background-color: white;
            padding: 20px;
            border-radius: 5px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
        }

        label {
            display: block;
            margin: 10px 0 5px;
        }

        input[type="file"] {
            display: block;
            margin-bottom: 20px;
        }

        input[type="submit"] {
            background-color: #4CAF50;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 5px;
            cursor: pointer;
        }

        input[type="submit"]:hover {
            background-color: #45a049;
        }

        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }

        table,
        th,
        td {
            border: 1px solid #ddd;
        }

        th,
        td {
            padding: 12px;
            text-align: left;
        }

        th {
            background-color: #f2f2f2;
        }

        #resultados {
            margin-top: 20px;
        }
    </style>
</head>

<body>
    <header>
        <h1>Comparação de Respostas</h1>
    </header>
    <main>
        <form id="upload-form">
            <label for="tabela1">Formulários dos veteranos:</label>
            <input type="file" id="tabela1" name="tabela1" accept=".csv" required>

            <label for="tabela2">Formulários dos bixos:</label>
            <input type="file" id="tabela2" name="tabela2" accept=".csv" required>

            <input type="submit" value="Comparar">
        </form>
        <div id="resultados"></div>
    </main>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.3.0/papaparse.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.3.1/jspdf.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.14/jspdf.plugin.autotable.min.js"></script>

    <script>
        document.getElementById('upload-form').addEventListener('submit', function (event) {
            event.preventDefault();
            const tabela1 = document.getElementById('tabela1').files[0];
            const tabela2 = document.getElementById('tabela2').files[0];
            if (!tabela1 || !tabela2) {
                alert("Por favor, selecione ambos os arquivos .csv.");
                return;
            }
            Promise.all([parseCSV(tabela1), parseCSV(tabela2)])
                .then(([data1, data2]) => {
                    console.log(data1);
                    console.log(data2);
                    if (data1.length === 0 || data2.length === 0) {
                        alert("Um dos arquivos .csv está vazio ou não pôde ser lido.");
                        return;
                    }
                    if (!data1[0].hasOwnProperty('Nome completo') || !data2[0].hasOwnProperty('Nome completo')) {
                        alert("Um dos arquivos .csv não contém a coluna 'Nome completo'.");
                        return;
                    }
                    const resultados = compararTabelas(data1, data2);
                    exibirResultados(resultados);
                    salvarResultadosPDF(resultados);
                    if (resultados.length > 0) {
                        marcarPareados(data1, data2, resultados);
                        if (dadosModificados(data1, tabela1)) {
                            salvarCSV(data1, 'tabela1_atualizada.csv');
                        }
                        if (dadosModificados(data2, tabela2)) {
                            salvarCSV(data2, 'tabela2_atualizada.csv');
                        }
                    }
                });
        });

        function parseCSV(file) {
            return new Promise((resolve, reject) => {
                Papa.parse(file, {
                    header: true,
                    complete: results => resolve(results.data),
                    error: err => reject(err)
                });
            });
        }

        function calcularDistanciaEuclidiana(respostas1, respostas2) {
            const somaQuadrados = respostas1.reduce((acc, val, index) => {
                const diferenca = parseInt(val) - parseInt(respostas2[index]);
                return acc + (diferenca * diferenca);
            }, 0);
            return Math.sqrt(somaQuadrados);
        }

        function extrairRespostasEscala(pessoa) {
            return Object.keys(pessoa)
                .filter(key => key !== 'Carimbo de data/hora' && key !== 'Nome completo')
                .map(key => pessoa[key]);
        }

        function compararTabelas(tabela1, tabela2) {
            const resultados = [];
            tabela1.forEach(pessoa1 => {
                if (pessoa1['Pareado'] !== 'Sim') {
                    tabela2.forEach(pessoa2 => {
                        if (pessoa2['Pareado'] !== 'Sim') {
                            const respostas1 = extrairRespostasEscala(pessoa1);
                            const respostas2 = extrairRespostasEscala(pessoa2);
                            const distancia = calcularDistanciaEuclidiana(respostas1, respostas2);
                            resultados.push({
                                pessoaTabela1: pessoa1['Nome completo'],
                                pessoaTabela2: pessoa2['Nome completo'],
                                distancia: distancia,
                                horario1: new Date(pessoa1['Carimbo de data/hora']),
                                horario2: new Date(pessoa2['Carimbo de data/hora'])
                            });
                        }
                    });
                }
            });
            resultados.sort((a, b) => a.distancia - b.distancia);
            const melhoresResultados = [];
            const usadosTabela1 = new Set();
            const usadosTabela2 = new Set();
            for (const resultado of resultados) {
                if (!usadosTabela1.has(resultado.pessoaTabela1) && !usadosTabela2.has(resultado.pessoaTabela2)) {
                    melhoresResultados.push(resultado);
                    usadosTabela1.add(resultado.pessoaTabela1);
                    usadosTabela2.add(resultado.pessoaTabela2);
                }
                if (melhoresResultados.length >= 10) break;
            }
            return melhoresResultados;
        }

        function exibirResultados(resultados) {
            const resultadosDiv = document.getElementById('resultados');
            resultadosDiv.innerHTML = '';
            if (resultados.length === 0) {
                resultadosDiv.textContent = "Nenhum resultado encontrado.";
                return;
            }
            const tabela = document.createElement('table');
            const thead = document.createElement('thead');
            const tbody = document.createElement('tbody');
            const cabecalho = ['Veteranos', 'Bixos', 'Distância Euclidiana'];
            const tr = document.createElement('tr');
            cabecalho.forEach(text => {
                const th = document.createElement('th');
                th.textContent = text;
                tr.appendChild(th);
            });
            thead.appendChild(tr);
            tabela.appendChild(thead);
            resultados.forEach(resultado => {
                const tr = document.createElement('tr');
                ['pessoaTabela1', 'pessoaTabela2', 'distancia'].forEach(key => {
                    const td = document.createElement('td');
                    td.textContent = resultado[key] || "N/A";
                    tr.appendChild(td);
                });
                tbody.appendChild(tr);
            });
            tabela.appendChild(tbody);
            resultadosDiv.appendChild(tabela);
        }

        function marcarPareados(tabela1, tabela2, resultados) {
            resultados.forEach(resultado => {
                const pessoa1 = tabela1.find(p => p['Nome completo'] === resultado.pessoaTabela1);
                const pessoa2 = tabela2.find(p => p['Nome completo'] === resultado.pessoaTabela2);
                if (pessoa1) pessoa1['Pareado'] = 'Sim';
                if (pessoa2) pessoa2['Pareado'] = 'Sim';
            });
        }

        function dadosModificados(data, original) {
            if (data.length !== original.length) return true;
            for (let i = 0; i < data.length; i++) {
                for (let key in data[i]) {
                    if (data[i][key] !== original[i][key]) {
                        return true;
                    }
                }
            }
            return false;
        }

        function salvarCSV(data, filename) {
            const csv = Papa.unparse(data);
            const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
            const link = document.createElement('a');
            link.href = URL.createObjectURL(blob);
            link.download = filename;
            link.style.display = 'none';
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        }

        function salvarResultadosPDF(resultados) {
            const { jsPDF } = window.jspdf;
            const doc = new jsPDF();

            const dadosTabela = resultados.map(resultado => [
                resultado.pessoaTabela1,
                resultado.pessoaTabela2,
                resultado.distancia.toFixed(2)
            ]);

            doc.autoTable({
                head: [['Veteranos', 'Bixos', 'Distância Euclidiana']],
                body: dadosTabela
            });

            doc.save('resultados_pareamento.pdf');
        }
    </script>

</body>

</html>