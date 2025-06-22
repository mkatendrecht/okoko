# okoko

const fromAssetSelect = document.getElementById('from-asset');
const toAssetSelect = document.getElementById('to-asset');

// New UI helpers for live price display
const amountInput     = document.getElementById('amount');
const fromPriceSmall  = document.getElementById('from-price');
const toPriceSmall    = document.getElementById('to-price');
const amountUsdSmall  = document.getElementById('amount-usd');

// Choices instances for searchable dropdowns
let fromChoices = null;
let toChoices   = null;

// Deep-link generators for popular DEX / bridge front-ends
const deepLinkGenerators = {
    '1inch': (r) => `https://app.1inch.io/#/1/swap/${r.fromToken.address}/${r.toToken.address}`,
    'uniswap_v3': (r) => `https://app.uniswap.org/#/swap?inputCurrency=${r.fromToken.address}&outputCurrency=${r.toToken.address}`,
    'uniswap_v2': (r) => `https://app.uniswap.org/#/swap?inputCurrency=${r.fromToken.address}&outputCurrency=${r.toToken.address}`,
    'sushiswap': (r) => `https://app.sushi.com/swap?inputCurrency=${r.fromToken.address}&outputCurrency=${r.toToken.address}`,
    'pancakeswap': (r) => `https://pancakeswap.finance/swap?inputCurrency=${r.fromToken.address}&outputCurrency=${r.toToken.address}`,
    'hop': (r) => `https://app.hop.exchange/#/send?token=${r.fromToken.symbol}&amount=${parseFloat(r.fromAmount)/(10**r.fromToken.decimals)}`
};

// How many alternative routes to request/display
const MAX_ROUTE_COUNT = 10;

// Cache of token metadata keyed by address { priceUsd, chainId, decimals, symbol, name }
const tokenMeta = {};

// Helper that refreshes the <small> elements according to current selections
function updatePriceDisplays() {
    const fromPrice = tokenMeta[fromAssetSelect.value]?.priceUsd;
    const toPrice   = tokenMeta[toAssetSelect.value]?.priceUsd;

    fromPriceSmall.textContent = fromPrice ? `Current price: $${parseFloat(fromPrice).toFixed(2)}` : 'Current price: N/A';
    toPriceSmall.textContent   = toPrice   ? `Current price: $${parseFloat(toPrice).toFixed(2)}`   : 'Current price: N/A';

    const amt = parseFloat(amountInput.value);
    if (!isNaN(amt) && fromPrice) {
        const usd = (amt * parseFloat(fromPrice)).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 });
        amountUsdSmall.textContent = `≈ $${usd}`;
    } else {
        amountUsdSmall.textContent = '';
    }
}

// Keep displays up-to-date as the user interacts
fromAssetSelect.addEventListener('change', updatePriceDisplays);
toAssetSelect.addEventListener('change', updatePriceDisplays);
amountInput.addEventListener('input', updatePriceDisplays);

// Function to fetch tokens and populate dropdowns
async function populateTokens() {
    try {
        // LI.FI API endpoint for tokens
        const response = await axios.get('https://li.quest/v1/tokens');
        // We are interested in tokens on Ethereum, which has chainId 1
        const tokens = response.data.tokens['1'];

        // Clear existing options
        fromAssetSelect.innerHTML = '';
        toAssetSelect.innerHTML = '';

        tokens.forEach(token => {
            const option = document.createElement('option');
            option.value = token.address;

            const price = token.priceUSD ? parseFloat(token.priceUSD).toFixed(2) : null;
            option.textContent = `${token.symbol} - ${token.name}`;

            option.dataset.chainId  = token.chainId;
            if (price) option.dataset.priceUsd = price; // keep for small text display

            // cache metadata
            tokenMeta[token.address] = {
                priceUsd: price,
                chainId: token.chainId,
                decimals: token.decimals,
                symbol: token.symbol,
                name: token.name
            };

            fromAssetSelect.appendChild(option.cloneNode(true));
            toAssetSelect.appendChild(option);
        });

        // Set default values to something that exists on Ethereum via LI.FI
        // For example, WETH and USDC
        fromAssetSelect.value = '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2'; // WETH
        toAssetSelect.value   = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48'; // USDC

        // Initialize or update the searchable dropdowns
        if (!fromChoices) {
            fromChoices = new Choices(fromAssetSelect, { searchEnabled: true, itemSelectText: '' });
            toChoices   = new Choices(toAssetSelect,   { searchEnabled: true, itemSelectText: '' });
        } else {
            // Rebuild the option lists inside Choices to keep them in sync
            const choiceData = tokens.map(t => {
                // update cache each refresh
                tokenMeta[t.address] = {
                    priceUsd: t.priceUSD ? parseFloat(t.priceUSD).toFixed(2) : null,
                    chainId: t.chainId,
                    decimals: t.decimals,
                    symbol: t.symbol,
                    name: t.name
                };
                return {
                    value: t.address,
                    label: `${t.symbol} - ${t.name}`
                };
            });
            fromChoices.setChoices(choiceData, 'value', 'label', true);
            toChoices.setChoices(choiceData,   'value', 'label', true);
        }

        // Immediately reflect prices in the UI
        updatePriceDisplays();

        // Update "Prices updated" timestamp in header
        const tsEl = document.querySelector('.updated-timestamp');
        if (tsEl) {
            tsEl.innerHTML = `<i class="fas fa-sync-alt"></i> Prices updated: ${new Date().toLocaleTimeString()}`;
        }

    } catch (error) {
        console.error('Error fetching tokens from LI.FI:', error);
    }
}

async function getLiFiRoutes(fromAsset, toAsset, amount, fromChainId, toChainId) {
    // Find token decimals for amount conversion
    let fromTokenDecimals;
    try {
        // A bit inefficient to fetch all tokens again, but good for this example
        const tokensResponse = await axios.get('https://li.quest/v1/tokens');
        const tokenList = tokensResponse.data.tokens[fromChainId];
        const fromToken = tokenList.find(token => token.address === fromAsset);
        if (!fromToken) throw new Error('From token not found');
        fromTokenDecimals = fromToken.decimals;
    } catch (error) {
        console.error("Could not get token decimals", error);
        return;
    }

    const amountInSmallestUnit = (amount * Math.pow(10, fromTokenDecimals)).toFixed(0);

    const routesRequest = {
        fromChainId: fromChainId,
        toChainId: toChainId,
        fromTokenAddress: fromAsset,
        toTokenAddress: toAsset,
        fromAmount: amountInSmallestUnit,
        options: {
            slippage: 0.005, // 0.5%
        }
    };

    // Build endpoint with explicit route count to ensure LI.FI returns up to MAX_ROUTE_COUNT
    const endpoint = `https://li.quest/v1/advanced/routes?maxRouteCount=${MAX_ROUTE_COUNT}`;

    try {
        const response = await axios.post(endpoint, routesRequest);
        console.log(response.data);
        return response.data;
    } catch (error) {
        console.error('Error fetching routes from LI.FI:', error);
    }
}

document.getElementById('transaction-form').addEventListener('submit', async function(event) {
    event.preventDefault();

    const analyzeBtn = this.querySelector('button[type="submit"]');
    const resultsContainer = document.getElementById('results-container');
    const resultsTableBody = document.querySelector('#results-table tbody');
    
    // Start loading state
    analyzeBtn.disabled = true;
    analyzeBtn.textContent = 'Analyzing...';
    resultsTableBody.innerHTML = '<tr><td colspan="6" style="text-align: center;">Fetching best options from LI.FI...</td></tr>';
    resultsContainer.style.display = 'block';


    const fromAsset = fromAssetSelect.value;
    const toAsset = toAssetSelect.value;
    // get both chainIds
    const fromChainId = Number(
        fromAssetSelect.options[fromAssetSelect.selectedIndex].dataset.chainId
    );
    const toChainId   = Number(
        toAssetSelect.options[toAssetSelect.selectedIndex].dataset.chainId
    );
    const amount = document.getElementById('amount').value;
    
    // show what we're about to analyse, even while loading
    const resultsSummary = document.getElementById('results-summary');
    resultsSummary.textContent =
        `Analyzing ${amount} ${tokenMeta[fromAsset].symbol} → `
    + `${tokenMeta[toAsset].symbol} …`;

    // call the API
    const routesResponse = await getLiFiRoutes(
        fromAsset,
        toAsset,
        amount,
        fromChainId,
        toChainId,
    );

    // if it comes back empty, update the summary too
    if (!routesResponse || !routesResponse.routes?.length) {
        resultsSummary.textContent =
            `No available routes for ${tokenMeta[fromAsset].symbol} → `
        + `${tokenMeta[toAsset].symbol} on the selected chains.`;
        resultsTableBody.innerHTML = '<tr><td colspan="6" style="text-align: center; color: red;">Error fetching routes from LI.FI or no routes found. Please try again.</td></tr>';
        return;
    }

    const { routes } = routesResponse;
    const bestRoute = routes[0]; // LI.FI sorts routes by best output by default

    resultsSummary.textContent = `Live analysis for ${amount} ${tokenMeta[fromAsset].symbol} → ${tokenMeta[toAsset].symbol} • showing top ${routes.length}/${MAX_ROUTE_COUNT} routes`;

    resultsTableBody.innerHTML = ''; // Clear loading/previous results

    // Update table header for LI.FI data
    document.querySelector('#results-table thead tr').innerHTML = `
        <th>Platform (via LI.FI)</th>
        <th>Est. Output</th>
        <th>Fees</th>
        <th>Gas (USD)</th>
        <th>Included Steps</th>
        <th>Swap</th>
    `;

    routes.forEach(route => {
        const toToken = route.toToken;
        const outputAmount = route.toAmount / Math.pow(10, toToken.decimals);

        // ----- fees & gas (handle multiple possible shapes) -----
        let totalFeesUSD = 0;
        let totalFeePercent = 0;
        if (Array.isArray(route.feeCosts)) {
            totalFeesUSD = route.feeCosts.reduce((total, fee) => total + parseFloat(fee.amountUSD || 0), 0);
            totalFeePercent = route.feeCosts.reduce((total, fee) => total + (fee.percentage ? parseFloat(fee.percentage) : 0), 0);
        } else if (route.feeCostUSD || route.feesUSD) {
            totalFeesUSD = parseFloat(route.feeCostUSD || route.feesUSD);
        }
        
        const feeDisplay = totalFeePercent > 0 
            ? `${(totalFeePercent * 100).toFixed(2)}% ($${totalFeesUSD.toFixed(2)})`
            : `$${totalFeesUSD.toFixed(2)}`;

        let totalGasUSD = 0;
        if (Array.isArray(route.gasCosts)) {
            totalGasUSD = route.gasCosts.reduce((total, gas) => total + parseFloat(gas.amountUSD || 0), 0);
        } else if (route.gasCostUSD) {
            totalGasUSD = parseFloat(route.gasCostUSD);
        }
        totalGasUSD = totalGasUSD.toFixed(2);

        // ----- step / tool names -----
        const cleanedStepNames = [];
        for (const step of route.steps) {
            const label = step.toolDetails?.name || step.tool || step.type || 'N/A';
            if (/^li\.fi/i.test(label)) continue;
            cleanedStepNames.push(label);
        }
        const stepNames = cleanedStepNames.join(' → ');

        if (cleanedStepNames.length === 0) return;

        // Pick first external tool as the platform label. If we don't have a direct
        // deep link, label it as Jumper Exchange since that's where the user will land.
        const firstStepToolKey = (route.steps[0].toolDetails?.key || route.steps[0].tool || '').toLowerCase();
        let primaryToolName = cleanedStepNames[0] || 'Unknown';

        if (!deepLinkGenerators[firstStepToolKey]) {
            primaryToolName = 'Jumper Exchange';
        }

        // ----- row output -----
        const row = document.createElement('tr');
        row.innerHTML = `
            <td>${primaryToolName}</td>
            <td>${outputAmount.toFixed(6)} ${toToken.symbol}</td>
            <td>${feeDisplay}</td>
            <td>$${totalGasUSD}</td>
            <td>${stepNames}</td>
            <td><a href="${getSwapUrl(route)}" target="_blank">Swap</a></td>
        `;
        resultsTableBody.appendChild(row);
    });


    resultsContainer.style.display = 'block';
});

function getSwapUrl(route) {
    const key = route.steps[0].toolDetails?.key || route.steps[0].tool || '';
    const gen = deepLinkGenerators[key.toLowerCase?.() || key];
    if (gen) {
        try { return gen(route); } catch { /* ignore */ }
    }
    // fall back to LI.FI Jumper with all params preserved
    return `https://jumper.exchange/?fromChain=${route.fromChainId}&fromToken=${route.fromToken.address}&toChain=${route.toChainId}&toToken=${route.toToken.address}&fromAmount=${route.fromAmount}`;
}

// Initial population of tokens and start 90-second refresh cycle
populateTokens();

// Refresh token list (and therefore prices) every 90 seconds
setInterval(() => {
    populateTokens();
}, 90_000); 
