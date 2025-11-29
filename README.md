<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>美股记账本</title>
    <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body { background-color: #F3F4F6; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
        .card-shadow { box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06); }
        .click-feedback:active { transform: scale(0.98); opacity: 0.8; }
        /* Hide scrollbar for clean UI */
        ::-webkit-scrollbar { width: 0px; background: transparent; }
        [v-cloak] { display: none; }
    </style>
</head>
<body>
    <div id="app" v-cloak class="max-w-md mx-auto min-h-screen bg-gray-100 pb-24 relative">
        
        <!-- 顶部资产卡片 -->
        <div class="bg-blue-600 text-white p-6 rounded-b-3xl shadow-lg relative z-10">
            <div class="flex justify-between items-start mb-4">
                <div>
                    <div class="text-blue-100 text-sm mb-1">总资产 (USD)</div>
                    <div class="text-3xl font-bold">${{ formatNumber(totalEquity) }}</div>
                </div>
                <div class="text-right cursor-pointer" @click="showChartModal = true">
                    <div class="text-blue-100 text-sm mb-1 flex items-center justify-end gap-1">
                        总盈亏 <span class="text-xs opacity-70">(点击查看曲线)</span>
                    </div>
                    <div class="text-xl font-semibold" :class="totalProfit >= 0 ? 'text-red-300' : 'text-green-300'">
                        {{ totalProfit >= 0 ? '+' : '' }}{{ formatNumber(totalProfit) }}
                    </div>
                </div>
            </div>
            
            <div class="grid grid-cols-2 gap-4 pt-4 border-t border-blue-500/30">
                <div>
                    <div class="text-blue-200 text-xs">可用资金</div>
                    <div class="font-medium">${{ formatNumber(cashBalance) }}</div>
                </div>
                <div class="text-right">
                    <div class="text-blue-200 text-xs">持仓市值</div>
                    <div class="font-medium">${{ formatNumber(marketValue) }}</div>
                </div>
            </div>
        </div>

        <!-- 设置初始资金入口 (如果没有交易记录且资金为默认值时显示) -->
        <div v-if="transactions.length === 0" class="m-4 p-4 bg-white rounded-xl card-shadow">
            <div class="text-sm text-gray-500 mb-2">设置初始本金</div>
            <div class="flex gap-2">
                <input type="number" v-model.number="initialCapital" class="flex-1 border rounded px-3 py-2" @change="saveData">
            </div>
        </div>

        <!-- 功能区 -->
        <div class="px-4 mt-4 flex justify-between items-center">
            <h3 class="font-bold text-gray-700">当前持仓</h3>
            <button @click="refreshPrices" :disabled="loadingPrices" class="text-xs bg-white text-blue-600 px-3 py-1 rounded-full border border-blue-200 shadow-sm active:bg-gray-50">
                {{ loadingPrices ? '更新中...' : '刷新行情' }}
            </button>
        </div>

        <!-- 持仓列表 -->
        <div class="p-4 space-y-3">
            <div v-if="holdings.length === 0" class="text-center text-gray-400 py-8 text-sm">
                暂无持仓，点击底部 "+" 记一笔
            </div>
            
            <div v-for="h in holdings" :key="h.symbol" 
                 @click="quickTrade(h)"
                 class="bg-white p-4 rounded-xl card-shadow click-feedback cursor-pointer flex justify-between items-center">
                <div>
                    <div class="font-bold text-lg">{{ h.symbol }}</div>
                    <div class="text-xs text-gray-400 mt-1">
                        持仓 {{ h.quantity }} 股
                        <span class="mx-1">|</span>
                        成本 ${{ formatNumber(h.avgPrice) }}
                    </div>
                </div>
                <div class="text-right">
                    <div class="font-bold text-lg">${{ formatNumber(h.currentPrice) }}</div>
                    <div class="text-xs font-medium" :class="h.unrealizedPL >= 0 ? 'text-red-500' : 'text-green-500'">
                        {{ h.unrealizedPL >= 0 ? '+' : '' }}{{ formatNumber(h.unrealizedPL) }}
                    </div>
                </div>
            </div>
        </div>

        <!-- 底部导航 -->
        <div class="fixed bottom-0 left-0 right-0 bg-white border-t border-gray-200 pb-safe flex justify-around items-center h-16 z-40 max-w-md mx-auto">
            <button @click="currentTab = 'home'" class="flex flex-col items-center justify-center w-full h-full" :class="currentTab === 'home' ? 'text-blue-600' : 'text-gray-400'">
                <span class="text-xl">📊</span>
                <span class="text-[10px]">资产</span>
            </button>
            
            <button @click="openTradeModal()" class="flex flex-col items-center justify-center w-full h-full -mt-6">
                <div class="w-12 h-12 bg-blue-600 rounded-full flex items-center justify-center text-white text-2xl shadow-lg shadow-blue-600/40 click-feedback">
                    +
                </div>
            </button>
            
            <button @click="currentTab = 'history'" class="flex flex-col items-center justify-center w-full h-full" :class="currentTab === 'history' ? 'text-blue-600' : 'text-gray-400'">
                <span class="text-xl">📝</span>
                <span class="text-[10px]">记录</span>
            </button>
        </div>

        <!-- 交易历史侧边栏/页面 -->
        <div v-if="currentTab === 'history'" class="fixed inset-0 bg-gray-100 z-30 pt-4 pb-20 overflow-y-auto max-w-md mx-auto">
             <div class="px-4 mb-4">
                <h2 class="text-xl font-bold text-gray-800">交易记录</h2>
                <div class="text-xs text-gray-500 mt-1">点击记录可修改或删除</div>
            </div>
            <div class="space-y-2 px-4">
                <div v-for="t in sortedTransactions" :key="t.id" @click="editTransaction(t)"
                     class="bg-white p-3 rounded-lg border border-gray-100 shadow-sm flex justify-between items-center active:bg-gray-50">
                    <div class="flex items-center gap-3">
                        <div class="w-1 h-8 rounded-full" :class="t.type === 'buy' ? 'bg-red-500' : 'bg-green-500'"></div>
                        <div>
                            <div class="font-bold text-gray-800">{{ t.symbol }}</div>
                            <div class="text-xs text-gray-400">{{ t.date }}</div>
                        </div>
                    </div>
                    <div class="text-right">
                        <div class="font-medium" :class="t.type === 'buy' ? 'text-red-600' : 'text-green-600'">
                            {{ t.type === 'buy' ? '买入' : '卖出' }} {{ t.quantity }}
                        </div>
                        <div class="text-xs text-gray-500">@ ${{ formatNumber(t.price) }} (费:{{ formatNumber(t.fee) }})</div>
                        <div v-if="t.type === 'sell' && t.realizedPL !== undefined" class="text-xs font-bold mt-1" :class="t.realizedPL >= 0 ? 'text-red-500' : 'text-green-500'">
                            盈亏: {{ t.realizedPL >= 0 ? '+' : '' }}{{ formatNumber(t.realizedPL) }}
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- 记账弹窗 -->
        <div v-if="showTradeModal" class="fixed inset-0 z-50 flex items-end justify-center bg-black/50 backdrop-blur-sm animate-fade-in">
            <div class="bg-white w-full max-w-md rounded-t-2xl p-6 animate-slide-up">
                <div class="flex justify-between items-center mb-6">
                    <h3 class="text-lg font-bold">{{ isEditing ? '编辑记录' : '记一笔' }}</h3>
                    <button @click="closeTradeModal" class="text-gray-400 text-2xl">&times;</button>
                </div>

                <div class="bg-gray-100 p-1 rounded-lg flex mb-6">
                    <button @click="form.type = 'buy'" :class="form.type === 'buy' ? 'bg-white shadow text-red-500' : 'text-gray-500'" class="flex-1 py-2 rounded-md text-sm font-bold transition-all">买入</button>
                    <button @click="form.type = 'sell'" :class="form.type === 'sell' ? 'bg-white shadow text-green-500' : 'text-gray-500'" class="flex-1 py-2 rounded-md text-sm font-bold transition-all">卖出</button>
                </div>

                <div class="space-y-4">
                    <div class="flex gap-4">
                        <div class="flex-1">
                            <label class="block text-xs text-gray-500 mb-1">股票代码</label>
                            <input v-model="form.symbol" @blur="form.symbol = form.symbol.toUpperCase()" type="text" placeholder="AAPL" class="w-full bg-gray-50 border border-gray-200 rounded-lg p-3 font-mono uppercase">
                        </div>
                        <div class="flex-1">
                            <label class="block text-xs text-gray-500 mb-1">日期</label>
                            <input v-model="form.date" type="date" class="w-full bg-gray-50 border border-gray-200 rounded-lg p-3">
                        </div>
                    </div>

                    <div class="flex gap-4">
                        <div class="flex-1">
                            <label class="block text-xs text-gray-500 mb-1">价格 ($)</label>
                            <input v-model.number="form.price" type="number" step="0.01" placeholder="0.00" class="w-full bg-gray-50 border border-gray-200 rounded-lg p-3">
                        </div>
                        <div class="flex-1">
                            <label class="block text-xs text-gray-500 mb-1">数量</label>
                            <input v-model.number="form.quantity" type="number" placeholder="0" class="w-full bg-gray-50 border border-gray-200 rounded-lg p-3">
                        </div>
                    </div>

                    <div class="bg-gray-50 rounded-lg p-3 flex justify-between items-center text-sm">
                        <span class="text-gray-500">手续费 (自动计算)</span>
                        <span class="font-mono">${{ calculatedFee }}</span>
                    </div>
                </div>

                <div class="mt-8 flex gap-3">
                    <button v-if="isEditing" @click="deleteTransaction" class="flex-1 bg-red-50 text-red-500 py-3 rounded-xl font-bold">删除</button>
                    <button @click="saveTransaction" class="flex-1 bg-blue-600 text-white py-3 rounded-xl font-bold shadow-lg shadow-blue-600/30">
                        {{ isEditing ? '保存修改' : '确认记账' }}
                    </button>
                </div>
            </div>
        </div>

        <!-- 盈利曲线图表弹窗 -->
        <div v-if="showChartModal" class="fixed inset-0 z-50 flex items-center justify-center bg-black/80 backdrop-blur-sm p-4" @click.self="showChartModal = false">
            <div class="bg-white w-full max-w-md rounded-2xl p-4 relative">
                <button @click="showChartModal = false" class="absolute top-4 right-4 text-gray-400 text-xl">&times;</button>
                <h3 class="text-lg font-bold mb-4">总盈亏走势</h3>
                <div class="h-64">
                    <canvas id="profitChart"></canvas>
                </div>
            </div>
        </div>

    </div>

    <script>
        const { createApp, ref, computed, watch, nextTick, onMounted } = Vue;

        createApp({
            setup() {
                // 状态
                const API_KEY = 'DTQP3SCGL3Y2W65O';
                const currentTab = ref('home');
                const showTradeModal = ref(false);
                const showChartModal = ref(false);
                const loadingPrices = ref(false);
                const isEditing = ref(false);
                const editingId = ref(null);

                const initialCapital = ref(10000.00);
                const transactions = ref([]);
                const priceMap = ref({}); // symbol -> price
                const profitHistory = ref([]); // [{date, profit}]

                // 表单数据
                const form = ref({
                    type: 'buy',
                    symbol: '',
                    date: new Date().toISOString().split('T')[0],
                    price: '',
                    quantity: '',
                    fee: 0
                });

                // 计算属性: 手续费
                const calculatedFee = computed(() => {
                    if (!form.value.price || !form.value.quantity) return '0.00';
                    const amount = form.value.price * form.value.quantity;
                    if (form.value.type === 'buy') {
                        return '20.00';
                    } else {
                        // 卖出: 20 + 0.3%
                        return (20 + (amount * 0.003)).toFixed(2);
                    }
                });

                // 计算属性: 排序后的交易记录 (带盈亏计算)
                const sortedTransactions = computed(() => {
                    // 1. 先按时间正序模拟一遍，计算每笔卖出的盈亏
                    const tempMap = {};
                    const enriched = transactions.value.map(t => ({...t})).sort((a, b) => new Date(a.date) - new Date(b.date) || a.id - b.id);
                    
                    enriched.forEach(t => {
                        if (!tempMap[t.symbol]) tempMap[t.symbol] = { quantity: 0, totalCost: 0 };
                        const h = tempMap[t.symbol];

                        if (t.type === 'buy') {
                            h.quantity += t.quantity;
                            h.totalCost += (t.price * t.quantity) + t.fee;
                        } else {
                            // 卖出
                            if (h.quantity > 0) {
                                const avgCost = h.totalCost / h.quantity;
                                // 卖出盈亏 = (卖出价 - 平均成本) * 数量 - 卖出手续费
                                const revenue = t.price * t.quantity;
                                const costBasis = avgCost * t.quantity;
                                t.realizedPL = revenue - costBasis - t.fee;
                                
                                // 更新剩余持仓成本
                                h.quantity -= t.quantity;
                                h.totalCost -= costBasis; 
                            } else {
                                t.realizedPL = 0; // 异常数据(卖空?)暂不计算
                            }
                        }
                    });

                    // 2. 倒序返回用于显示
                    return enriched.sort((a, b) => new Date(b.date) - new Date(a.date) || b.id - a.id);
                });

                // 核心逻辑: 从交易记录推导持仓
                const holdings = computed(() => {
                    const map = {};
                    
                    // 按时间正序处理
                    const sorted = [...transactions.value].sort((a, b) => new Date(a.date) - new Date(b.date));
                    
                    sorted.forEach(t => {
                        if (!map[t.symbol]) map[t.symbol] = { quantity: 0, totalCost: 0 };
                        
                        if (t.type === 'buy') {
                            map[t.symbol].quantity += t.quantity;
                            // 成本包含手续费
                            map[t.symbol].totalCost += (t.price * t.quantity) + t.fee;
                        } else {
                            const oldQty = map[t.symbol].quantity;
                            map[t.symbol].quantity -= t.quantity;
                            // 卖出按比例减少成本 (简单平均法)
                            if (oldQty > 0) {
                                const costPerShare = map[t.symbol].totalCost / oldQty;
                                map[t.symbol].totalCost -= (costPerShare * t.quantity);
                            }
                        }
                    });

                    // 转换为数组并计算盈亏
                    return Object.keys(map)
                        .filter(sym => map[sym].quantity > 0) // 只显示有持仓的
                        .map(sym => {
                            const h = map[sym];
                            const currentPrice = priceMap.value[sym] || (h.totalCost / h.quantity); // 没价格时暂用成本价
                            const marketValue = h.quantity * currentPrice;
                            // 持仓盈亏 = 市值 - 剩余成本
                            const unrealizedPL = marketValue - h.totalCost;
                            return {
                                symbol: sym,
                                quantity: h.quantity,
                                avgPrice: h.totalCost / h.quantity,
                                currentPrice: currentPrice,
                                unrealizedPL: unrealizedPL,
                                marketValue: marketValue
                            };
                        });
                });

                // 资产计算
                const marketValue = computed(() => {
                    return holdings.value.reduce((sum, h) => sum + h.marketValue, 0);
                });

                const cashBalance = computed(() => {
                    let cash = initialCapital.value;
                    transactions.value.forEach(t => {
                        const amount = t.price * t.quantity;
                        if (t.type === 'buy') {
                            cash -= (amount + t.fee);
                        } else {
                            cash += (amount - t.fee);
                        }
                    });
                    return cash;
                });

                const totalEquity = computed(() => cashBalance.value + marketValue.value);
                const totalProfit = computed(() => totalEquity.value - initialCapital.value);

                // 方法
                const formatNumber = (num) => {
                    if (num === undefined || num === null || isNaN(num)) return '0.00';
                    return num.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
                };

                const loadData = () => {
                    const saved = localStorage.getItem('stock_app_v2');
                    if (saved) {
                        const data = JSON.parse(saved);
                        initialCapital.value = data.initialCapital;
                        transactions.value = data.transactions;
                        priceMap.value = data.priceMap || {};
                        profitHistory.value = data.profitHistory || [];
                    }
                };

                const saveData = () => {
                    // 记录每日快照
                    const today = new Date().toISOString().split('T')[0];
                    const lastSnapshot = profitHistory.value[profitHistory.value.length - 1];
                    
                    const snapshot = { date: today, profit: totalProfit.value };
                    
                    if (lastSnapshot && lastSnapshot.date === today) {
                        profitHistory.value[profitHistory.value.length - 1] = snapshot;
                    } else {
                        profitHistory.value.push(snapshot);
                    }

                    localStorage.setItem('stock_app_v2', JSON.stringify({
                        initialCapital: initialCapital.value,
                        transactions: transactions.value,
                        priceMap: priceMap.value,
                        profitHistory: profitHistory.value
                    }));
                };

                const fetchPrice = async (symbol) => {
                    try {
                        const res = await fetch(`https://www.alphavantage.co/query?function=GLOBAL_QUOTE&symbol=${symbol}&apikey=${API_KEY}`);
                        const data = await res.json();
                        const quote = data['Global Quote'];
                        if (quote && quote['05. price']) {
                            priceMap.value[symbol] = parseFloat(quote['05. price']);
                        }
                    } catch (e) {
                        console.error(e);
                    }
                };

                const refreshPrices = async () => {
                    loadingPrices.value = true;
                    const symbols = [...new Set(holdings.value.map(h => h.symbol))];
                    for (const sym of symbols) {
                        await fetchPrice(sym);
                        // 简单的限流延迟
                        await new Promise(r => setTimeout(r, 1000));
                    }
                    loadingPrices.value = false;
                    saveData();
                };

                const openTradeModal = () => {
                    isEditing.value = false;
                    form.value = {
                        type: 'buy',
                        symbol: '',
                        date: new Date().toISOString().split('T')[0],
                        price: '',
                        quantity: '',
                        fee: 0
                    };
                    showTradeModal.value = true;
                };

                const quickTrade = (holding) => {
                    openTradeModal();
                    form.value.symbol = holding.symbol;
                    form.value.price = holding.currentPrice;
                };

                const editTransaction = (t) => {
                    isEditing.value = true;
                    editingId.value = t.id;
                    form.value = { ...t };
                    showTradeModal.value = true;
                };

                const saveTransaction = () => {
                    if (!form.value.symbol || !form.value.price || !form.value.quantity) return;
                    
                    const fee = parseFloat(calculatedFee.value);
                    const transaction = {
                        id: isEditing.value ? editingId.value : Date.now(),
                        ...form.value,
                        symbol: form.value.symbol.toUpperCase(),
                        fee: fee
                    };

                    if (isEditing.value) {
                        const index = transactions.value.findIndex(t => t.id === editingId.value);
                        transactions.value[index] = transaction;
                    } else {
                        transactions.value.push(transaction);
                    }
                    
                    saveData();
                    closeTradeModal();
                    // 如果是新股，尝试获取价格
                    if (!priceMap.value[transaction.symbol]) {
                        fetchPrice(transaction.symbol);
                    }
                };

                const deleteTransaction = () => {
                    if (!confirm('确定要删除这条记录吗？后续持仓将重新计算。')) return;
                    transactions.value = transactions.value.filter(t => t.id !== editingId.value);
                    saveData();
                    closeTradeModal();
                };

                const closeTradeModal = () => {
                    showTradeModal.value = false;
                };

                // Chart.js logic
                let chart = null;
                watch(showChartModal, (val) => {
                    if (val) {
                        nextTick(() => {
                            const ctx = document.getElementById('profitChart').getContext('2d');
                            if (chart) chart.destroy();
                            
                            chart = new Chart(ctx, {
                                type: 'line',
                                data: {
                                    labels: profitHistory.value.map(h => h.date),
                                    datasets: [{
                                        label: '累计盈亏 (USD)',
                                        data: profitHistory.value.map(h => h.profit),
                                        borderColor: '#2563EB',
                                        backgroundColor: 'rgba(37, 99, 235, 0.1)',
                                        fill: true,
                                        tension: 0.4
                                    }]
                                },
                                options: {
                                    responsive: true,
                                    maintainAspectRatio: false,
                                    interaction: { intersect: false, mode: 'index' },
                                    scales: {
                                        y: { grid: { borderDash: [4, 4] } },
                                        x: { grid: { display: false } }
                                    }
                                }
                            });
                        });
                    }
                });

                // 自动尝试获取价格 (只在首次加载且有持仓时)
                onMounted(() => {
                    loadData();
                    if (holdings.value.length > 0 && Object.keys(priceMap.value).length === 0) {
                        refreshPrices();
                    }
                });

                return {
                    currentTab,
                    initialCapital,
                    transactions,
                    holdings,
                    marketValue,
                    cashBalance,
                    totalEquity,
                    totalProfit,
                    sortedTransactions,
                    showTradeModal,
                    showChartModal,
                    loadingPrices,
                    form,
                    isEditing,
                    calculatedFee,
                    formatNumber,
                    saveData,
                    refreshPrices,
                    openTradeModal,
                    quickTrade,
                    editTransaction,
                    saveTransaction,
                    deleteTransaction,
                    closeTradeModal
                };
            }
        }).mount('#app');
    </script>
</body>
</html>
