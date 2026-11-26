const crypto = require('crypto');

const appId = $env.SHOPEE_APP_ID;
const secret = $env.SHOPEE_SECRET;

if (!appId || !secret) {
  throw new Error('Defina SHOPEE_APP_ID e SHOPEE_SECRET no arquivo .env antes de executar o fluxo.');
}

const ENDPOINT = 'https://open-api.affiliate.shopee.com.br/graphql';
const MAX_TENTATIVAS = 5;   // quantas buscas tentar até achar um produto
const QTD = 1;              // produtos por execução
const MIN_NOTA = 4.5;
const CHANCE_NATAL = 0.4;   // 0.4 = 40% dos envios usam as categorias de Natal
const USAR_SUBIDS = true;   // false = envia o link longo, sem rastreio por subId

// comissão em decimal: 0.08 = 8%
const FAIXAS = [
  { ate: 100,      minComissao: 0.07, minVendas: 300 },
  { ate: 300,      minComissao: 0.06, minVendas: 150 },
  { ate: 1000,     minComissao: 0.05, minVendas: 50  },
  { ate: Infinity, minComissao: 0.03, minVendas: 20  }
];

// ===== CATEGORIAS (peso maior = aparece mais) =====
const CATEGORIAS = {
  casa: { peso: 2, termos: [
    'cortina blackout', 'organizador acrílico geladeira', 'mop giratório', 'rodo mágico',
    'jogo de cama', 'toalha de banho', 'luminária led'
  ] },
  cozinha: { peso: 1, termos: [
    'chaleira elétrica', 'mini processador de alimentos', 'pote hermético',
    'escorredor de louça', 'balança digital de cozinha', 'garrafa térmica', 'utensílios de cozinha'
  ] },
  beleza: { peso: 2, termos: [
    'sérum vitamina c', 'protetor solar bastão', 'pimple patch', 'óleo essencial difusor',
    'secador de cabelo', 'hidratante corporal', 'kit pincéis maquiagem'
  ] },
  modaFeminina: { peso: 2, termos: [
    'calça wide leg', 'calça jeans feminina', 'sandália feminina', 'calcinha sem costura',
    'bolsa transversal', 'vestido midi', 'pijama feminino'
  ] },
  modaMasculina: { peso: 1, termos: [
    'kit relógio masculino', 'carteira masculina', 'kit meias', 'chinelo slide',
    'mochila impermeável', 'óculos de sol', 'bota coturno masculina'
  ] },
  pet: { peso: 2, termos: [
    'areia para gato', 'cama nuvem pet', 'comedouro elevado pet', 'brinquedo interativo gato',
    'arranhador para gato', 'tapete higiênico', 'coleira peitoral'
  ] },
  autoMoto: { peso: 2, termos: [
    'retrovisor moto', 'pastilha de freio moto', 'suporte celular moto', 'capa de chuva moto',
    'luva motociclista', 'aromatizante carro', 'aspirador de pó automotivo'
  ] },
  tecnologia: { peso: 1, termos: [
    'caixa de som bluetooth', 'fone bluetooth', 'smartwatch amoled', 'câmera wifi',
    'ring light', 'fita led', 'capinha de celular'
  ] },
  ferramentas: { peso: 1, termos: [
    'lanterna recarregável', 'kit de ferramentas', 'parafusadeira', 'trena a laser',
    'lanterna de cabeça', 'caixa de ferramentas', 'alicate'
  ] },
  fitness: { peso: 1, termos: [
    'tapete yoga', 'mini band', 'corda de pular', 'luva de treino',
    'garrafa academia 2 litros', 'pistola massageadora', 'legging academia'
  ] },
  infantil: { peso: 1, termos: [
    'brinquedo educativo', 'bloco de montar', 'boia inflável infantil', 'roupinha de bebê',
    'mochila infantil', 'carrinho controle remoto', 'quebra-cabeça'
  ] },
  ticketAlto: { peso: 1, termos: [
    'smart tv', 'air fryer', 'aspirador robô', 'notebook', 'geladeira',
    'cadeira gamer', 'colchão', 'máquina de lavar'
  ] }
};

const CATEGORIAS_NATAL = {
  natalDecoracao: { peso: 2, termos: [
    'árvore de natal', 'bola de natal', 'guirlanda de natal', 'enfeite de natal',
    'papai noel decoração', 'boneco de neve decoração', 'presépio'
  ] },
  natalLuzes: { peso: 1, termos: [
    'pisca-pisca natal', 'cortina de led natal', 'luz de natal', 'projetor de natal',
    'árvore de natal led', 'estrela de natal luz', 'fita led natal'
  ] },
  natalMesa: { peso: 1, termos: [
    'toalha de mesa natal', 'jogo americano natal', 'prato de natal', 'caminho de mesa natal',
    'porta guardanapo natal', 'centro de mesa natal', 'forma de panetone'
  ] },
  natalEmbrulho: { peso: 1, termos: [
    'papel de presente natal', 'caixa de presente natal', 'sacola de presente natal',
    'laço de natal', 'fita de cetim natal', 'etiqueta de presente natal', 'meia de natal'
  ] },
  natalRoupas: { peso: 1, termos: [
    'pijama de natal', 'blusa de natal', 'gorro de papai noel', 'camiseta natal família',
    'avental de natal', 'chinelo natal', 'vestido natal'
  ] },
  natalCasaPet: { peso: 1, termos: [
    'tapete de natal', 'capa de almofada natal', 'cortina de natal', 'jogo de cama natal',
    'toalha de banho natal', 'roupinha pet natal', 'fantasia pet natal'
  ] }
};

// ===== FUNÇÕES =====
const faixaDe = (preco) => FAIXAS.find(f => preco < f.ate);
const sorteia = (lista) => lista[Math.floor(Math.random() * lista.length)];
const espera = (ms) => new Promise(r => setTimeout(r, ms));
const limpa = (s) => String(s).normalize('NFD').replace(/[\u0300-\u036f]/g, '')
  .toLowerCase().replace(/[^a-z0-9]/g, '').slice(0, 20);

// sorteia uma categoria respeitando os pesos
const sorteiaCategoria = (grupo) => {
  const nomes = Object.keys(grupo);
  const total = nomes.reduce((soma, n) => soma + (grupo[n].peso || 1), 0);
  let r = Math.random() * total;
  for (const n of nomes) {
    r -= (grupo[n].peso || 1);
    if (r <= 0) return n;
  }
  return nomes[nomes.length - 1];
};

const memoria = $getWorkflowStaticData('global');
memoria.enviados = memoria.enviados || [];

const hora = 'h' + new Intl.DateTimeFormat('pt-BR', {
  hour: '2-digit', hourCycle: 'h23', timeZone: 'America/Manaus'
}).format(new Date());

const CAMPOS_BASE = 'itemId productName priceMin priceMax ratingStar sales imageUrl shopName commissionRate offerLink';
let campos = CAMPOS_BASE + ' productLink';

// chamada assinada à API da Shopee
const chamar = async (payload) => {
  const timestamp = Math.floor(Date.now() / 1000);
  const signature = crypto.createHash('sha256')
    .update(appId + timestamp + payload + secret)
    .digest('hex');

  const resp = await this.helpers.httpRequest({
    method: 'POST',
    url: ENDPOINT,
    headers: {
      'Content-Type': 'application/json',
      Authorization: `SHA256 Credential=${appId}, Timestamp=${timestamp}, Signature=${signature}`
    },
    body: payload,
    json: false
  });
  return typeof resp === 'string' ? JSON.parse(resp) : resp;
};

const buscar = (keyword, page) => chamar(JSON.stringify({
  query: `{
    productOfferV2(keyword: "${keyword}", sortType: 2, page: ${page}, limit: 50) {
      nodes { ${campos} }
    }
  }`
}));

// gera o link curto com subIds; devolve null se falhar
const gerarLink = async (origem, subIds) => {
  const o = JSON.stringify(origem);
  const s = JSON.stringify(subIds);
  const formas = [
    `mutation { generateShortLink(input: {originUrl: ${o}, subIds: ${s}}) { shortLink } }`,
    `mutation { generateShortLink(originUrl: ${o}, subIds: ${s}) { shortLink } }`
  ];
  for (const query of formas) {
    try {
      const r = await chamar(JSON.stringify({ query }));
      const link = r?.data?.generateShortLink?.shortLink;
      if (link) return link;
    } catch (e) { /* tenta a próxima forma */ }
  }
  return null;
};

// ===== BUSCA COM TENTATIVAS =====
let ultimoErro = '';
let errouAntes = false;

for (let tentativa = 1; tentativa <= MAX_TENTATIVAS; tentativa++) {
  const usouNatal = Math.random() < CHANCE_NATAL;
  const grupo = usouNatal ? CATEGORIAS_NATAL : CATEGORIAS;
  const categoria = sorteiaCategoria(grupo);
  const keyword = sorteia(grupo[categoria].termos);
  const page = errouAntes ? 1 : Math.floor(Math.random() * 3) + 1;

  let res;
  try {
    res = await buscar(keyword, page);
  } catch (e) {
    ultimoErro = `${keyword} (pág ${page}): ${e.message}`;
    errouAntes = true;
    await espera(2000);
    continue;
  }

  if (res.errors) {
    const txt = JSON.stringify(res.errors);
    if (txt.includes('productLink')) campos = CAMPOS_BASE;   // campo não aceito: tenta sem ele
    ultimoErro = `${keyword} (pág ${page}): ${txt}`;
    errouAntes = true;
    await espera(2000);
    continue;
  }
  errouAntes = false;

  const nodes = res?.data?.productOfferV2?.nodes || [];

  const novos = nodes
    .map(n => {
      const preco = Number(n.priceMin);
      const taxa = Number(n.commissionRate);
      return { ...n, _preco: preco, _taxa: taxa, _ganho: preco * taxa };
    })
    .filter(n => n._preco > 0 && n._taxa > 0)
    .filter(n => {
      const f = faixaDe(n._preco);
      return n._taxa >= f.minComissao && Number(n.sales) >= f.minVendas;
    })
    .filter(n => Number(n.ratingStar) >= MIN_NOTA)
    .filter(n => !memoria.enviados.includes(n.itemId))
    .sort((a, b) => Number(b.sales) - Number(a.sales))
    .slice(0, QTD);

  if (novos.length === 0) continue;   // nenhum serviu, tenta outra busca

  memoria.enviados.push(...novos.map(n => n.itemId));
  memoria.enviados = memoria.enviados.slice(-1000);

  const saida = [];
  for (const p of novos) {
    const faixa = faixaDe(p._preco);
    const subIds = [
      'whatsapp',
      hora,
      limpa(keyword),
      limpa(categoria),
      faixa.ate === Infinity ? 'acima1000' : 'ate' + faixa.ate
    ];

    let link = p.offerLink;
    let linkTipo = 'longo';
    if (USAR_SUBIDS && p.productLink) {
      const curto = await gerarLink(p.productLink, subIds);
      if (curto) { link = curto; linkTipo = 'curto'; }
    }

    const preco = p._preco.toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' });
    const texto = [
      `🔥 *${p.productName}*`,
      `💰 ${preco}`,
      `⭐ ${p.ratingStar} | 🛒 ${p.sales} vendidos`,
      ``,
      `👉 ${link}`
    ].join('\n');

    saida.push({ json: {
      texto,
      imagem: p.imageUrl,
      ganhoEstimado: Number(p._ganho.toFixed(2)),
      linkTipo,
      subIds: subIds.join(',')
    } });
  }
  return saida;
}

// se a última tentativa terminou em erro da API, a execução aparece como falha
if (errouAntes) {
  throw new Error(`Shopee falhou em ${MAX_TENTATIVAS} tentativas. Último erro: ${ultimoErro}`);
}
return [];   // nenhuma busca achou produto que passasse nos filtros