<div class="mx-auto flex flex-col h-screen">
    <div class="text-lg italic bg-yellow-200 mb-1 flex flex-row px-1 flex-none">
        <div class="flex-1">Расчёт налогов на вклады (сроком на 1 год)</div>
        <div class="flex-none">
            <a
                href="https://www.nalog.ru/rn77/news/activities_fts/10007632/"
                rel="noreferrer"
                target="_blank"
                class="hover:underline"
            >Правила расчёта</a>
        </div>
    </div>

    <div class="flex-1">
        {#each groups as group}
        <div class="flex flex-row bg-purple-200 mb-1">
            <div class="flex-1 p-1">
                <div class="flex flex-row">
                    <div class="flex-none pr-1">Название группы:</div>
                    <input bind:value={group.name} class="flex-1 px-1"/>
                    <span on:click={removeGroup(group)} class="cursor-pointer text-red-900 hover:text-red-500 pl-1">удалить</span>
                </div>

                <div class="mt-1">
                    Базовая сумма: <input type="number" bind:value={group.baseAmount} class="w-28 px-1" />
                    Ставка ЦБ: <input bind:value={group.bankRate} class="w-10 px-1" />%
                    Налоги: <input bind:value={group.taxRate} class="w-10 px-1" />%
                    Мин. процент: <input bind:value={group.minRate} class="w-10 px-1" />%
                </div>

                <h2 class="text-xl font-bold">Вклады</h2>
                {#each group.items as item}
                <div class="mb-1">
                    Сумма: <input type="number" bind:value={item.amount} class="w-28 px-1"/>
                    Проценты: <input bind:value={item.rate} class="w-10 px-1"/>%
                    <span on:click={removeItem(group, item)} class="cursor-pointer text-red-900 hover:text-red-500">удалить</span>
                </div>
                {/each}

                <button class="text-green-900 text-lg px-1" on:click={addItem(group)}>Добавить вклад</button>
            </div>

            <div class="flex-none p-1">
                <table>
                    <tbody>
                        {#each groupRows(group) as row}
                        <tr>
                            <td class="font-bold text-right">{row[0]}:</td>
                            <td class="text-xl">{row[1]}</td>
                        </tr>
                        {/each}
                    </tbody>
                </table>
            </div>
        </div>
        {/each}

        <div>
            <button class="text-green-900 text-lg px-1" on:click={addGroup}>Добавить группу</button>
        </div>
    </div>

    <div class="flex-none mt-3 px-1 text-center text-blue-300">
        🄯
        <span class="italic">
            <a class="hover:underline" href="https://alkatrazstudio.net/" rel="noreferrer" target="_blank">Alkatraz Studio</a>, 2020
        </span>
        |
        <span class="italic">
            <a class="hover:underline" href="https://github.com/alkatrazstudio/nalogi" rel="noreferrer" target="_blank">исходники</a>
        </span>
    </div>
</div>

<script>
import { onMount } from 'svelte'
import localforage from 'localforage'
import cloneDeep from 'lodash/cloneDeep'

const DEFAULT_BANK_RATE = 4.25
const DEFAULT_BASE_AMOUNT = 1_000_000
const DEFAULT_TAX_RATE = 13
const DEFAULT_MIN_RATE = 1
const DEFAULT_AMOUNT = 1_000_000
const DEFAULT_RATE = 6

export let groups = []
export let allowSaving = false

export function addGroup()
{
    const lastGroup = cloneDeep(groups[groups.length - 1])

    groups = [...groups, {
        bankRate: DEFAULT_BANK_RATE,
        baseAmount: DEFAULT_BASE_AMOUNT,
        taxRate: DEFAULT_TAX_RATE,
        minRate: DEFAULT_MIN_RATE,
        items: [],
        ...(lastGroup || []),
        name: `Группа ${groups.length + 1}`
    }]

    if(!lastGroup)
        addItem(groups[groups.length - 1])
}

export function addItem(group)
{
    group.items = [...group.items, {
        amount: DEFAULT_AMOUNT,
        rate: DEFAULT_RATE
    }]
    groups = groups
    allowSaving = true
}

export function removeItem(group, item)
{
    group.items = group.items.filter(i => i !== item)
    groups = groups
}

export function removeGroup(group)
{
    groups = groups.filter(i => i !== group)
}

async function loadConfig()
{
    try{
        groups = await localforage.getItem('groups') || []
    }catch(e){
        groups = []
    }
    if(!groups.length)
        addGroup()
}

async function saveConfig(groups)
{
    await localforage.setItem('groups', groups)
}

const moneyFormatter = new Intl.NumberFormat('ru-RU', {
    style: 'currency',
    currency: 'RUB',
    minimumFractionDigits: 0,
    maximumFractionDigits: 0
})

const rateFormatter = new Intl.NumberFormat('ru-RU', {
    style: 'percent',
    minimumFractionDigits: 0,
    maximumFractionDigits: 2
})

export function m(x){return moneyFormatter.format(x)}
export function r(x){return rateFormatter.format(x)}
function i(x){return parseInt(x) || 0}
function f(x){return parseFloat(x) || 0}

const totalAmount = g => Math.ceil(g.items.reduce((s, x) => s + i(x.amount), 0))
const totalIncome = g => Math.ceil(g.items.reduce((s, x) => s + i(x.amount) * f(x.rate) / 100, 0))
const totalIncomeInTaxableAccounts = g => Math.ceil(g.items.reduce((s, x) => s + i(x.amount) * (f(x.rate) > f(g.minRate) ? f(x.rate) : 0) / 100, 0))
const avgIncomeRate = g => Math.ceil(totalIncome(g) / totalAmount(g) * 1000) / 1000 || 0
const nonTaxAmountLimit = g => Math.ceil(i(g.baseAmount) * f(g.bankRate) / 100)
const nonTaxAmount = g => Math.min(totalIncome(g), nonTaxAmountLimit(g))
const taxedIncome = g => Math.max(0, totalIncomeInTaxableAccounts(g) - nonTaxAmountLimit(g))
const taxedIncomeRatio = g => Math.ceil(10000 * taxedIncome(g) / totalIncome(g)) / 10000 || 0
const taxAmount = g => Math.ceil(taxedIncome(g) * f(g.taxRate) / 100)
const realIncome = g => totalIncome(g) - taxAmount(g)
const taxOfIncome = g => Math.ceil(taxAmount(g) / totalIncome(g) * 10000) / 10000 || 0
const realAvgIncomeRate = g => Math.ceil(10000 * realIncome(g) / totalAmount(g) ) / 10000 || 0
const avgIncomeRateLoss = g => avgIncomeRate(g) - realAvgIncomeRate(g)

export function groupRows(group)
{
    return [
        ['Общая сумма вкладов', m(totalAmount(group))],
        ['Общий доход (без налогов)', m(totalIncome(group))],
        ['Общий доход (без налогов) с облагаемых налогом вкладов', m(totalIncomeInTaxableAccounts(group))],
        ['Средний процент', r(avgIncomeRate(group))],
        ['Предел дохода, не облагаемого налогом', m(nonTaxAmountLimit(group))],
        ['Доход, не облагаемый налогом', m(nonTaxAmount(group))],
        ['Доход, облагаемый налогом', m(taxedIncome(group))],
        ['Процент дохода, облагаемого налогом', r(taxedIncomeRatio(group))],
        ['Налог', m(taxAmount(group))],
        ['Процент налога с дохода', r(taxOfIncome(group))],
        ['Итоговый доход с учётом налогов', m(realIncome(group))],
        ['Итоговый средний процент с учётом налогов', r(realAvgIncomeRate(group))],
        ['Потеря процентов из-за налогов', r(avgIncomeRateLoss(group))]
    ]
}

$: {
    if(allowSaving)
        saveConfig(groups)
}

onMount(loadConfig)
</script>
