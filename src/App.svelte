<script>
  const UNITS = ['g', 'kg', 'ml', 'L', '個', '枚', '本', '袋', '箱'];
  const DEFAULT_NAMES = ['商品A', '商品B', '商品C', '商品D', '商品E', '商品F', '商品G', '商品H'];

  let globalUnit = $state('g');
  let nextId = $state(4);
  let products = $state([
    { id: 1, name: '', price: '', amount: '', discount: '', showDiscount: false },
    { id: 2, name: '', price: '', amount: '', discount: '', showDiscount: false },
    { id: 3, name: '', price: '', amount: '', discount: '', showDiscount: false },
  ]);

  function placeholder(index) {
    return DEFAULT_NAMES[index] ?? `商品${index + 1}`;
  }

  function displayName(product, index) {
    return product.name.trim() || placeholder(index);
  }

  function effectivePrice(p) {
    const price = parseFloat(p.price);
    if (isNaN(price) || price < 0) return null;
    const disc = parseFloat(p.discount);
    const factor = !isNaN(disc) && disc > 0 && disc < 100 ? 1 - disc / 100 : 1;
    return price * factor;
  }

  let unitPrices = $derived(
    products.map((p) => {
      const ep = effectivePrice(p);
      const amount = parseFloat(p.amount);
      if (ep === null || isNaN(amount) || amount <= 0) return null;
      return ep / amount;
    })
  );

  let validPairs = $derived(
    products.map((p, i) => ({ p, i, price: unitPrices[i] })).filter((x) => x.price !== null)
  );

  let minPrice = $derived(
    validPairs.length > 0 ? Math.min(...validPairs.map((x) => x.price)) : null
  );

  let ranks = $derived.by(() => {
    if (validPairs.length === 0) return products.map(() => null);
    const sorted = [...new Set(validPairs.map((x) => x.price))].sort((a, b) => a - b);
    return unitPrices.map((p) => (p === null ? null : sorted.indexOf(p) + 1));
  });

  let savings = $derived.by(() => {
    if (minPrice === null) return products.map(() => null);
    return unitPrices.map((p) => {
      if (p === null || p === minPrice) return null;
      return ((p - minPrice) / minPrice) * 100;
    });
  });

  function fmt(p) {
    if (p === null) return null;
    if (p < 0.001) return `¥${p.toExponential(2)}`;
    if (p < 0.01)  return `¥${p.toFixed(5)}`;
    if (p < 0.1)   return `¥${p.toFixed(4)}`;
    if (p < 1)     return `¥${p.toFixed(3)}`;
    if (p < 10)    return `¥${p.toFixed(2)}`;
    if (p < 1000)  return `¥${p.toFixed(1)}`;
    return `¥${Math.round(p).toLocaleString('ja-JP')}`;
  }

  function addProduct() {
    products = [
      ...products,
      { id: nextId++, name: '', price: '', amount: '', discount: '', showDiscount: false },
    ];
  }

  function removeProduct(id) {
    if (products.length <= 2) return;
    products = products.filter((p) => p.id !== id);
  }

  const RANK_LABELS = ['最安値', '2位', '3位', '4位', '5位', '6位', '7位', '8位'];
  function rankLabel(r) {
    return RANK_LABELS[r - 1] ?? `${r}位`;
  }
</script>

<div class="app">
  <header>
    <div class="header-inner">
      <h1>単価比較ツール</h1>
      <p>価格・内容量を入力するとリアルタイムに単価を計算・比較します</p>
    </div>
  </header>

  <main>
    <!-- 単位セレクター -->
    <div class="unit-bar">
      <span class="unit-label">単位</span>
      <div class="unit-pills">
        {#each UNITS as u}
          <button class="pill" class:pill-active={globalUnit === u} onclick={() => (globalUnit = u)}>
            {u}
          </button>
        {/each}
      </div>
    </div>

    <div class="grid">
      {#each products as product, i (product.id)}
        {@const np = unitPrices[i]}
        {@const rank = ranks[i]}
        {@const saving = savings[i]}
        {@const priceStr = fmt(np)}
        {@const ep = effectivePrice(product)}
        {@const hasDiscount = ep !== null && parseFloat(product.price) !== ep}
        <div class="card" class:is-best={rank === 1 && np !== null}>

          <!-- 商品名 + 削除ボタン -->
          <div class="card-top">
            <input
              class="name-input"
              type="text"
              placeholder={placeholder(i)}
              bind:value={product.name}
              aria-label="商品名"
            />
            {#if products.length > 2}
              <button class="btn-remove" onclick={() => removeProduct(product.id)} aria-label="削除">
                ×
              </button>
            {/if}
          </div>

          <!-- モバイル専用: 単価を別行で表示（card-topから分離してオーバーフロー回避） -->
          <div class="result-mobile">
            {#if np !== null}
              <span class="rm-price" class:best-color={rank === 1}>{priceStr}</span>
              <span class="rm-unit">/{globalUnit}</span>
              {#if hasDiscount}
                <span class="rm-discount">(-{product.discount}%)</span>
              {/if}
              {#if rank === 1}
                <span class="badge badge-best">最安値</span>
              {:else if saving !== null}
                <span class="badge badge-diff">+{saving.toFixed(1)}%</span>
              {/if}
            {:else}
              <span class="rm-empty">価格と内容量を入力</span>
            {/if}
          </div>

          <!-- デスクトップ専用: 単価を大きく表示 -->
          <div class="price-area">
            {#if np !== null}
              <div class="price-val" class:best-color={rank === 1}>{priceStr}</div>
              <div class="price-unit">
                /{globalUnit}
                {#if hasDiscount}<span class="discount-note">(-{product.discount}%適用)</span>{/if}
              </div>
              <div class="badge-row">
                {#if rank === 1}
                  <span class="badge badge-best">{rankLabel(1)}</span>
                {:else if rank !== null}
                  <span class="badge badge-rank">{rankLabel(rank)}</span>
                  {#if saving !== null}
                    <span class="badge badge-diff">最安値より +{saving.toFixed(1)}%</span>
                  {/if}
                {/if}
              </div>
            {:else}
              <div class="price-empty">—</div>
              <div class="price-hint">価格と内容量を入力</div>
            {/if}
          </div>

          <!-- 入力フィールド -->
          <div class="inputs">
            <div class="field">
              <label for="price-{product.id}">価格</label>
              <div class="input-row">
                <span class="affix">¥</span>
                <input
                  id="price-{product.id}"
                  type="number"
                  min="0"
                  step="any"
                  placeholder="198"
                  inputmode="decimal"
                  bind:value={product.price}
                />
              </div>
            </div>
            <div class="field">
              <label for="amount-{product.id}">内容量</label>
              <div class="input-row">
                <input
                  id="amount-{product.id}"
                  type="number"
                  min="0.001"
                  step="any"
                  placeholder="100"
                  inputmode="decimal"
                  bind:value={product.amount}
                />
                <span class="affix affix-right">{globalUnit}</span>
              </div>
            </div>
          </div>

          <!-- 値引き（オプション） -->
          {#if product.showDiscount}
            <div class="discount-field">
              <label for="disc-{product.id}">値引き</label>
              <div class="input-row discount-row">
                <input
                  id="disc-{product.id}"
                  type="number"
                  min="0"
                  max="99"
                  step="any"
                  placeholder="0"
                  inputmode="decimal"
                  bind:value={product.discount}
                />
                <span class="affix affix-right">%</span>
              </div>
            </div>
          {/if}

          <button
            class="btn-discount"
            onclick={() => {
              product.showDiscount = !product.showDiscount;
              if (!product.showDiscount) product.discount = '';
            }}
          >
            {product.showDiscount ? '値引きをキャンセル' : '+ 値引きを追加'}
          </button>
        </div>
      {/each}
    </div>

    <div class="add-area">
      <button class="btn-add" onclick={addProduct}>
        <span class="plus-icon">+</span> 商品を追加
      </button>
    </div>

    <!-- 比較サマリー -->
    {#if validPairs.length >= 2}
      <section class="summary">
        <h2>比較結果</h2>
        <div class="summary-rows">
          {#each [...validPairs].sort((a, b) => (ranks[a.i] ?? 99) - (ranks[b.i] ?? 99)) as { p, i, price }}
            {@const r = ranks[i]}
            {@const sav = savings[i]}
            <div class="summary-row" class:summary-best={r === 1}>
              <span
                class="s-rank"
                class:s-rank-1={r === 1}
                class:s-rank-2={r === 2}
                class:s-rank-3={r === 3}
                class:s-rank-other={r !== null && r > 3}
              >{rankLabel(r)}</span>
              <span class="s-name">{displayName(p, i)}</span>
              <span class="s-price">{fmt(price)}/{globalUnit}</span>
              {#if sav !== null}
                <span class="s-diff">+{sav.toFixed(1)}%</span>
              {:else}
                <span class="s-check">最安値</span>
              {/if}
            </div>
          {/each}
        </div>
      </section>
    {/if}
  </main>

  <footer>
    単価 = 価格 ÷ 内容量 で計算します（値引きは価格に反映）
  </footer>
</div>

<style>
  :global(*, *::before, *::after) { box-sizing: border-box; margin: 0; padding: 0; }
  :global(html) { font-size: 16px; }
  :global(body) {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Hiragino Sans',
      'Hiragino Kaku Gothic ProN', Meiryo, sans-serif;
    background: #eef1f7;
    color: #1e293b;
    min-height: 100vh;
    /* モバイルでの横スクロール防止 */
    overflow-x: hidden;
  }
  /* number input のスピンボタンを非表示（スマホで幅を食わないよう） */
  :global(input[type='number']::-webkit-inner-spin-button),
  :global(input[type='number']::-webkit-outer-spin-button) {
    -webkit-appearance: none;
    margin: 0;
  }
  :global(input[type='number']) { -moz-appearance: textfield; }

  .app { min-height: 100vh; display: flex; flex-direction: column; }

  /* ── Header ── */
  header {
    background: linear-gradient(135deg, #4f6ef7 0%, #8b5cf6 100%);
    color: white;
    padding: 2.25rem 1.5rem 2.5rem;
  }
  .header-inner { max-width: 1100px; margin: 0 auto; text-align: center; }
  header h1 { font-size: 1.85rem; font-weight: 800; letter-spacing: -0.03em; margin-bottom: 0.4rem; }
  header p { opacity: 0.82; font-size: 0.88rem; }

  /* ── Main ── */
  main {
    flex: 1;
    max-width: 1100px;
    /* width: 100% + overflow-x: hidden でカード幅を封じ込める */
    width: 100%;
    overflow-x: hidden;
    margin: -1.1rem auto 0;
    padding: 0 1rem 3rem;
  }

  /* ── Unit bar ── */
  .unit-bar {
    background: white;
    border-radius: 14px;
    box-shadow: 0 2px 12px rgba(0,0,0,0.07);
    padding: 0.75rem 1.1rem;
    margin-bottom: 1.1rem;
    display: flex;
    align-items: center;
    gap: 0.75rem;
    flex-wrap: wrap;
  }
  .unit-label { font-size: 0.78rem; font-weight: 700; color: #64748b; white-space: nowrap; }
  .unit-pills { display: flex; gap: 0.35rem; flex-wrap: wrap; }
  .pill {
    border: 1.5px solid #e2e8f0;
    background: #f8fafc;
    color: #475569;
    border-radius: 99px;
    padding: 0.25rem 0.7rem;
    font-size: 0.82rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.15s;
  }
  .pill:hover { border-color: #4f6ef7; color: #4f6ef7; }
  .pill-active { background: #4f6ef7; border-color: #4f6ef7; color: white; }

  /* ── Grid ── */
  .grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
    gap: 1rem;
    margin-bottom: 1.25rem;
  }

  /* ── Card ── */
  .card {
    background: white;
    border-radius: 18px;
    /* カード内コンテンツのはみ出しを防ぐ */
    min-width: 0;
    overflow: hidden;
    padding: 1.25rem;
    box-shadow: 0 2px 12px rgba(0,0,0,0.07);
    border: 2.5px solid transparent;
    transition: transform 0.15s, box-shadow 0.15s, border-color 0.2s;
  }
  .card:hover { transform: translateY(-3px); box-shadow: 0 8px 24px rgba(0,0,0,0.11); }
  .is-best { border-color: #22c55e; background: linear-gradient(160deg, #f0fdf4 0%, #fff 55%); }

  /* 商品名行 */
  .card-top { display: flex; align-items: center; gap: 0.4rem; margin-bottom: 0.75rem; }
  .name-input {
    flex: 1;
    /* flex アイテムのデフォルト min-width を潰す */
    min-width: 0;
    border: none;
    border-bottom: 2px solid #e2e8f0;
    background: transparent;
    font-size: 1rem;
    font-weight: 700;
    color: #1e293b;
    padding: 0.2rem 0;
    outline: none;
    transition: border-color 0.15s;
  }
  .name-input:focus { border-bottom-color: #4f6ef7; }
  .name-input::placeholder { color: #94a3b8; font-weight: 500; }
  .btn-remove {
    flex-shrink: 0;
    background: none; border: none; cursor: pointer;
    color: #94a3b8; font-size: 1.15rem;
    width: 28px; height: 28px;
    display: flex; align-items: center; justify-content: center;
    border-radius: 50%;
    transition: background 0.15s, color 0.15s;
  }
  .btn-remove:hover { background: #fee2e2; color: #ef4444; }

  /* ── モバイル専用 単価行（デフォルト非表示） ── */
  .result-mobile { display: none; }

  /* ── デスクトップ 単価エリア ── */
  .price-area {
    text-align: center;
    padding: 0.65rem 0 0.75rem;
    min-height: 105px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }
  .price-val {
    font-size: 2.3rem; font-weight: 800;
    letter-spacing: -0.03em; line-height: 1; color: #1e293b;
  }
  .best-color { color: #16a34a; }
  .price-unit { font-size: 0.82rem; color: #64748b; margin-top: 0.2rem; }
  .discount-note { font-size: 0.72rem; color: #f59e0b; margin-left: 0.3rem; }
  .price-empty { font-size: 2.3rem; color: #cbd5e1; font-weight: 300; line-height: 1; }
  .price-hint { font-size: 0.75rem; color: #94a3b8; margin-top: 0.3rem; }
  .badge-row {
    margin-top: 0.5rem;
    display: flex; gap: 0.3rem; justify-content: center; flex-wrap: wrap;
    min-height: 1.5rem;
  }

  /* Badges */
  .badge {
    display: inline-block;
    padding: 0.18rem 0.6rem; border-radius: 99px;
    font-size: 0.72rem; font-weight: 800; white-space: nowrap;
  }
  .badge-best { background: #22c55e; color: white; }
  .badge-rank { background: #e2e8f0; color: #475569; }
  .badge-diff { background: #fee2e2; color: #dc2626; }

  /* ── 入力フィールド ── */
  .inputs {
    display: flex;
    flex-direction: column;
    gap: 0.6rem;
    border-top: 1px solid #f1f5f9;
    padding-top: 0.9rem;
    margin-top: 0.35rem;
  }
  .field {
    display: flex; flex-direction: column; gap: 0.2rem;
    /* flex アイテムが縮めるよう min-width を潰す */
    min-width: 0;
  }
  .field label {
    font-size: 0.67rem; font-weight: 700; color: #94a3b8;
    text-transform: uppercase; letter-spacing: 0.08em;
  }
  .input-row {
    display: flex;
    border: 1.5px solid #e2e8f0;
    border-radius: 10px;
    overflow: hidden;
    transition: border-color 0.15s;
    /* 幅が親を超えないよう */
    min-width: 0;
  }
  .input-row:focus-within { border-color: #4f6ef7; }
  .affix {
    background: #f8fafc;
    padding: 0 0.55rem;
    border-right: 1.5px solid #e2e8f0;
    color: #64748b; font-size: 0.9rem;
    display: flex; align-items: center; flex-shrink: 0;
  }
  .affix-right {
    border-right: none;
    border-left: 1.5px solid #e2e8f0;
  }
  input[type='number'] {
    /* flex: 1 + min-width: 0 で親に収まるよう強制 */
    flex: 1;
    min-width: 0;
    width: 0; /* flex でのみ広がるよう */
    border: none; outline: none;
    padding: 0.5rem 0.65rem;
    font-size: 0.95rem;
    background: transparent; color: #1e293b;
  }

  /* ── 値引きフィールド ── */
  .discount-field {
    display: flex;
    align-items: center;
    gap: 0.6rem;
    margin-top: 0.6rem;
    padding-top: 0.6rem;
    border-top: 1px dashed #e2e8f0;
  }
  .discount-field label {
    font-size: 0.67rem; font-weight: 700; color: #f59e0b;
    text-transform: uppercase; letter-spacing: 0.08em;
    white-space: nowrap;
  }
  .discount-row { flex: 1; min-width: 0; }

  .btn-discount {
    display: block;
    width: 100%;
    margin-top: 0.6rem;
    background: none;
    border: none;
    color: #94a3b8;
    font-size: 0.75rem;
    cursor: pointer;
    text-align: left;
    padding: 0;
    transition: color 0.15s;
  }
  .btn-discount:hover { color: #4f6ef7; }

  /* ── 追加ボタン ── */
  .add-area { display: flex; justify-content: center; margin-bottom: 1.75rem; }
  .btn-add {
    display: inline-flex; align-items: center; gap: 0.4rem;
    background: linear-gradient(135deg, #4f6ef7, #8b5cf6);
    color: white; border: none; border-radius: 99px;
    padding: 0.7rem 2rem; font-size: 0.95rem; font-weight: 700;
    cursor: pointer;
    box-shadow: 0 3px 12px rgba(79,110,247,0.4);
    transition: transform 0.15s, box-shadow 0.15s;
  }
  .btn-add:hover { transform: translateY(-2px); box-shadow: 0 6px 20px rgba(79,110,247,0.5); }
  .btn-add:active { transform: none; }
  .plus-icon { font-size: 1.1rem; font-weight: 400; line-height: 1; }

  /* ── サマリー ── */
  .summary {
    background: white; border-radius: 18px;
    padding: 1.35rem 1.35rem 1.1rem;
    box-shadow: 0 2px 12px rgba(0,0,0,0.07);
  }
  .summary h2 {
    font-size: 0.78rem; font-weight: 800; color: #94a3b8;
    margin-bottom: 0.85rem; text-transform: uppercase; letter-spacing: 0.08em;
  }
  .summary-rows { display: flex; flex-direction: column; gap: 0.4rem; }
  .summary-row {
    display: flex; align-items: center; gap: 0.65rem;
    padding: 0.65rem 0.85rem; background: #f8fafc;
    border-radius: 10px; border: 1.5px solid transparent;
  }
  .summary-best { background: #f0fdf4; border-color: #22c55e; }
  .s-rank {
    flex-shrink: 0;
    padding: 0.15rem 0.55rem; border-radius: 99px;
    font-size: 0.7rem; font-weight: 800;
  }
  .s-rank-1 { background: #22c55e; color: white; }
  .s-rank-2 { background: #94a3b8; color: white; }
  .s-rank-3 { background: #b45309; color: white; }
  .s-rank-other { background: #e2e8f0; color: #475569; }
  .s-name {
    flex: 1; min-width: 0;
    font-weight: 600; color: #1e293b; font-size: 0.88rem;
    overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
  }
  .s-price { font-weight: 700; color: #1e293b; font-size: 0.88rem; white-space: nowrap; }
  .s-diff { font-size: 0.75rem; color: #dc2626; font-weight: 700; white-space: nowrap; }
  .s-check { font-size: 0.75rem; color: #16a34a; font-weight: 800; white-space: nowrap; }

  /* ── Footer ── */
  footer { background: #1e293b; color: #64748b; text-align: center; padding: 1rem; font-size: 0.8rem; }

  /* ── モバイル（〜640px） ── */
  @media (max-width: 640px) {
    header { padding: 1.5rem 1rem 1.75rem; }
    header h1 { font-size: 1.4rem; }
    header p { font-size: 0.82rem; }

    /* main のパディングを詰める */
    main { padding: 0 0.6rem 2rem; }

    .unit-bar { padding: 0.6rem 0.9rem; gap: 0.55rem; margin-bottom: 0.8rem; }

    .grid { grid-template-columns: 1fr; gap: 0.6rem; }

    .card {
      padding: 0.8rem;
      border-radius: 14px;
    }
    /* ホバーエフェクトはモバイルで無効（タップ時に浮かないよう） */
    .card:hover { transform: none; box-shadow: 0 2px 12px rgba(0,0,0,0.07); }

    /* デスクトップの大きな単価エリアを非表示 */
    .price-area { display: none; }

    /* モバイル単価行: card-top の下に独立した行として表示 */
    .result-mobile {
      display: flex;
      align-items: baseline;
      flex-wrap: wrap; /* 長い場合は折り返し */
      gap: 0.2rem 0.35rem;
      margin-bottom: 0.7rem;
    }
    .rm-price { font-size: 1.05rem; font-weight: 800; color: #1e293b; }
    .rm-unit  { font-size: 0.75rem; color: #64748b; }
    .rm-discount { font-size: 0.72rem; color: #f59e0b; }
    .rm-empty { font-size: 0.82rem; color: #94a3b8; }
    /* モバイルのバッジを少し小さく */
    .result-mobile .badge { font-size: 0.65rem; padding: 0.12rem 0.45rem; }

    /* 入力を横並び */
    .inputs { flex-direction: row; gap: 0.45rem; padding-top: 0.65rem; margin-top: 0; }
    /* 各フィールドの min-width を 0 にして flex で縮められるよう */
    .inputs .field { flex: 1; min-width: 0; }

    input[type='number'] { padding: 0.45rem 0.35rem; font-size: 0.88rem; }
    .affix { padding: 0 0.4rem; font-size: 0.82rem; }

    .discount-field { flex-wrap: wrap; }
    .discount-row { flex: 1; min-width: 120px; }

    .btn-discount { font-size: 0.72rem; margin-top: 0.5rem; }

    .summary { border-radius: 14px; padding: 1rem; }
    .summary-row { flex-wrap: wrap; gap: 0.4rem; }
  }
</style>
