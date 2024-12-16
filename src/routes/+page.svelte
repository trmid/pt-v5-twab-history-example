<script lang="ts">
  import { getTwabHistory } from "pt-v5-twab-history"
  import { createPublicClient, formatUnits, http, parseAbi, type Address, type PublicClient } from "viem"
  import { optimism, base, arbitrum, mainnet, scroll, worldchain, gnosis } from "viem/chains"

  const chains = [optimism, base, arbitrum, mainnet, scroll, worldchain, gnosis].sort((a,b) => a.name < b.name ? -1 : 1)
  const twabControllers: Record<number, { address: Address, offset: number }> = {
    [optimism.id]: {
      address: "0xCB0672dE558Ad8F122C0E081f0D35480aB3be167",
      offset: 1713398400
    },
    [base.id]: {
      address: "0x7e63601f7e28c758feccf8cdf02f6598694f44c6",
      offset: 1715799600
    },
    [arbitrum.id]: {
      address: "0x971ecc4e75c5fcfd8fc3eadc8f0c900b5914dc75",
      offset: 1717009200
    },
    [mainnet.id]: {
      address: "0x4d5f2cd31701f3e5de77b3f89ee7b80eb87b4acc",
      offset: 1724094000
    },
    [scroll.id]: {
      address: "0x5ec48e749768aea9956cb38542a9837ec714537d",
      offset: 1726016400
    },
    [worldchain.id]: {
      address: "0xBae945b19C6D9b306d18Da48501bf634B6A6813B",
      offset: 1733432400
    },
    [gnosis.id]: {
      address: "0x84090aea5370565b88108c4ffed378672a8afde6",
      offset: 1726016400
    }
  }

  let chain: typeof chains[0]
  let vaultAddress: string = ""
  let accountAddress: string | undefined
  let chartContainer: HTMLDivElement
  let querying = false

  const query = async () => {
    if (!chain || !vaultAddress) return
    querying = true
    try {
      const client = createPublicClient({ chain, transport: http(undefined, { batch: true }) })
      const [observations, decimals, symbol] = await Promise.all([
        getTwabHistory({
          viemClient: client as PublicClient,
          vault: vaultAddress as Address,
          account: accountAddress as Address | undefined,
          twabController: twabControllers[chain.id].address,
          twabOffset: twabControllers[chain.id].offset
        }),
        client.readContract({
          address: vaultAddress as Address,
          abi: parseAbi(["function decimals() external view returns (uint8)"]),
          functionName: "decimals"
        }),
        client.readContract({
          address: vaultAddress as Address,
          abi: parseAbi(["function symbol() external view returns (string)"]),
          functionName: "symbol"
        })
      ])
      console.log({ observations, decimals })

      const name = `${symbol}${accountAddress ? `:${accountAddress.slice(0, 6)}...${accountAddress.slice(-4)}` : ""}`
      const yAxisName = accountAddress ? "Delegate Balance" : "TVL"
      const chart = new CanvasJS.Chart("chartContainer", {
        //Chart Options - Check https://canvasjs.com/docs/charts/chart-options/
        height: 500,
        title:{
          text: `${yAxisName} History for ${name}`,
          fontFamily: "monospace",
          fontSize: "24"
        },
        axisY:[{
          title: yAxisName,
          fontFamily: "monospace",
          lineColor: "#4c249f",
          tickColor: "#4c249f",
          labelFontColor: "#4c249f",
          titleFontColor: "#4c249f",
          includeZero: true
        }],
        data: [{
          type: "stepLine",
          name,
          color: "#4c249f",
          fontFamily: "monospace",
          showInLegend: false,
          axisYIndex: 0,
          dataPoints: observations.map(x => ({
            x: new Date(x.timestamp * 1000),
            y: parseFloat(formatUnits(x.balance, decimals))
          }))
        }]
      });
      chart.render();
    } finally {
      querying = false
    }
  }
</script>

<main>
  <h1>PoolTogether V5 TWAB History</h1>
  <p>
    This tool queries the entire available observation history for vault TVL or user balance (including delegations) in PoolTogether V5.
  </p>
  <div class="inputs">
    <select bind:value={chain}>
      {#each chains as chain}
        <option value={chain}>{chain.name}</option>
      {/each}
    </select>
    <input type="text" name="vaultAddress" placeholder="Vault Address (0xabcd...)" bind:value={vaultAddress}>
    <input type="text" name="accountAddress" placeholder="User Address (optional)" bind:value={accountAddress}>
    <div>
      <button on:click={query} disabled={querying || !vaultAddress || !chain}>Query</button>
    </div>
  </div>
  <div bind:this={chartContainer} id="chartContainer"></div>
</main>
<footer>
  <p>Check out the <a href="https://www.npmjs.com/package/pt-v5-twab-history" target="_blank">pt-v5-twab-history</a> library to use this query in your own project!</p>
  <span>Made with 💜 by trmid.eth</span>
</footer>

<style>
  :global(html) {
    background-color: rgb(76, 36, 159);
    background: linear-gradient(rgb(76, 36, 159) 0, rgb(101, 56, 193) 100%);
    background-attachment: fixed;
    background-size: cover;
    color: #fff;
    font-family: monospace;
  }

  a, a:visited {
    color: #aaa;
  }

  main, footer {
    width: 1024px;
    margin: 0 auto;
    padding: 1rem;
  }

  .inputs {
    max-width: 500px;
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  input, select, button {
    padding: 0.5rem 1rem;
    border-radius: 0.5rem;
    border: none;
  }

  #chartContainer {
    margin-top: 1rem;
    height: 500px;
    border-radius: 0.5rem;
    overflow: hidden;
  }
</style>