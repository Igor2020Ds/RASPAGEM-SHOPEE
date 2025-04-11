# RASPAGEM-SHOPEE
Script rola a página no console do navegador e armazena dados como: nomeProduto, preco, imagem e demais dados de venda em arquivo excel
(async () => {
  const startTime = Date.now();
  const updateStatus = (msg) => {
    const now = ((Date.now() - startTime) / 1000).toFixed(1);
    statusDiv.textContent = `⏱ ${now}s | ${msg}`;
  };

  // Status visual
  const statusDiv = document.createElement('div');
  Object.assign(statusDiv.style, {
    position: 'fixed', top: '10px', left: '10px', padding: '10px 15px',
    background: '#000', color: '#0f0', fontSize: '14px', zIndex: 9999,
    fontFamily: 'monospace'
  });
  document.body.appendChild(statusDiv);

  updateStatus('Carregando bibliotecas...');

  const loadScript = (src) => new Promise(resolve => {
    const script = document.createElement('script');
    script.src = src;
    script.onload = resolve;
    document.head.appendChild(script);
  });

  await Promise.all([
    loadScript('https://cdn.jsdelivr.net/npm/exceljs@4.3.0/dist/exceljs.min.js'),
    loadScript('https://cdnjs.cloudflare.com/ajax/libs/FileSaver.js/2.0.5/FileSaver.min.js')
  ]);

  const workbook = new ExcelJS.Workbook();
  const sheet = workbook.addWorksheet('Produtos');

  sheet.columns = [
    { header: 'Nome', key: 'nome', width: 60 },
    { header: 'Preço', key: 'preco', width: 15 },
    { header: 'Loja', key: 'loja_nome', width: 30 },
    { header: 'Link', key: 'link', width: 10 },
    { header: 'Vendas Est. 30 dias', key: 'vendas_est', width: 20 }
  ];

  const produtos = [];
  const pesquisa = new URLSearchParams(location.search).get('keyword') || 'cachorro';
  let pagina = 0;
  const limitePorPagina = 60;
  let fim = false;

  // Cache para nomes de loja
  const nomesLoja = {};

  async function buscarNomeRealLoja(shopid) {
    if (nomesLoja[shopid]) return nomesLoja[shopid];
    try {
      const res = await fetch(`https://shopee.com.br/api/v4/shop/get_shop_detail?shopid=${shopid}`);
      const json = await res.json();
      const nome = json.data.account.username || `Loja ${shopid}`;
      nomesLoja[shopid] = nome;
      return nome;
    } catch {
      return `Loja ${shopid}`;
    }
  }

  while (!fim) {
    updateStatus(`Buscando página ${pagina + 1}...`);

    const url = `https://shopee.com.br/api/v4/search/search_items?by=relevancy&keyword=${pesquisa}&limit=${limitePorPagina}&newest=${pagina * limitePorPagina}&order=desc&page_type=search&scenario=PAGE_GLOBAL_SEARCH&version=2`;

    try {
      const res = await fetch(url);
      const json = await res.json();
      const items = json.items;

      if (!items || items.length === 0) {
        fim = true;
        break;
      }

      const batch = await Promise.all(items.map(async item => {
        const data = item.item_basic;
        const nome = data.name;
        const preco = (data.price / 100000).toFixed(2);
        const lojaId = data.shopid;
        const lojaNomeReal = await buscarNomeRealLoja(lojaId);
        const lojaLink = `https://shopee.com.br/shop/${lojaId}`;
        const vendas = Math.round((data.historical_sold || 0) * 0.3);
        const link = `https://shopee.com.br/${nome.replace(/[^a-zA-Z0-9]+/g, '-')}-i.${lojaId}.${data.itemid}`;
        const imagemURL = `https://down-br.img.susercontent.com/file/${data.image}`;
        return { nome, preco, lojaId, lojaNome: lojaNomeReal, lojaLink, link, imagemURL, vendas };
      }));

      produtos.push(...batch);
      pagina++;

    } catch (e) {
      console.error('Erro ao buscar página:', e);
      fim = true;
    }
  }

  updateStatus(`Baixando imagens e criando planilha (${produtos.length} produtos)...`);

  for (let i = 0; i < produtos.length; i++) {
    const p = produtos[i];

    sheet.addRow({
      nome: p.nome,
      preco: p.preco,
      loja_nome: { text: p.lojaNome, hyperlink: p.lojaLink },
      link: { text: 'Veja', hyperlink: p.link },
      vendas_est: p.vendas
    });

    try {
      const imgBlob = await fetch(p.imagemURL).then(r => r.blob());
      const arrayBuffer = await imgBlob.arrayBuffer();
      const imageId = workbook.addImage({ buffer: arrayBuffer, extension: 'jpeg' });

      sheet.addImage(imageId, {
        tl: { col: 5, row: i + 1 },
        ext: { width: 64, height: 64 }
      });

      sheet.getRow(i + 2).height = 52;
    } catch (e) {
      console.warn(`Erro ao baixar imagem: ${p.imagemURL}`);
    }

    const percent = Math.floor(((i + 1) / produtos.length) * 100);
    updateStatus(`Processando ${i + 1}/${produtos.length} (${percent}%)`);
  }

  updateStatus('Criando aba de resumo...');

  const resumoSheet = workbook.addWorksheet('Resumo');

  const vendasPorLoja = {};
  for (const p of produtos) {
    if (!vendasPorLoja[p.lojaId]) {
      vendasPorLoja[p.lojaId] = { total: 0, lojaNome: p.lojaNome, lojaLink: p.lojaLink };
    }
    vendasPorLoja[p.lojaId].total += p.vendas;
  }

  const lojasOrdenadas = Object.entries(vendasPorLoja)
    .sort((a, b) => b[1].total - a[1].total);

  resumoSheet.columns = [
    { header: 'Loja', key: 'loja', width: 40 },
    { header: 'Vendas Est. 30 dias', key: 'total', width: 20 }
  ];

  for (const [, { total, lojaNome, lojaLink }] of lojasOrdenadas) {
    resumoSheet.addRow({
      loja: { text: lojaNome, hyperlink: lojaLink },
      total
    });
  }

  updateStatus('Salvando Excel...');

  const buffer = await workbook.xlsx.writeBuffer();
  const blob = new Blob([buffer], {
    type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
  });
  saveAs(blob, `produtos_${pesquisa}_${produtos.length}.xlsx`);

  const finalTime = ((Date.now() - startTime) / 1000).toFixed(1);
  updateStatus(`✅ Finalizado em ${finalTime}s. Total: ${produtos.length} produtos.`);
})();
