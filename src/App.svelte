<script>
  /**********         IMPORTS         **********/
  import Chart from "chart.js/auto";
  import { onMount } from "svelte";
  /**********       END IMPORTS         **********/

  /**********         CONSTANTS         **********/

  const API_KEY = "2KMZ72MWUZ2MK96M";

  /**********         END CONSTANTS         **********/

  /**********        VARIABLE DEFINE         **********/
  /**
   * Is there a wrong input
   */
  let errorMessage = false;
  $: errorMessage;

  /**
   * Starter portfolio object
   */
  let portfolioObj = {
    stocks: {},
    cash: 0,
    customAsset: {},
  };
  let loaded = false;
  $: portfolioObj, updateChart();
  let stockObjGraph = {};
  let cashAmount = 0;
  let stockName = "";
  let stockAmount = 0;
  let stockMetrics = {};
  /**********         END VARIABLE DEFINE         **********/

  onMount(async () => {
    loaded = true;
    if (localStorage.getItem("portfolioObject") !== null) {
      portfolioObj = JSON.parse(localStorage.getItem("portfolioObject"));
    }
    if (localStorage.getItem("stockObjGraph") !== null) {
      stockObjGraph = JSON.parse(localStorage.getItem("stockObjGraph"));
    }
    if (localStorage.getItem("cashAmount") !== null) {
      cashAmount = Number(localStorage.getItem("cashAmount"));
    }
    if (localStorage.getItem("stockMetrics") !== null) {
      stockMetrics = JSON.parse(localStorage.getItem("stockMetrics"));
    }
  });

  const updatePortfolio = () => {
    // check for empty values
    if (stockName.length > 1 && stockAmount > 1) {
      // add to the stocks property of portfolioObject with a key of the inputted stock name and value being the amount
      portfolioObj["stocks"][stockName] = stockAmount;
    } else {
      // if there are empty values, error out
      console.log("HELLO?");
      errorMessage = true;
    }
    // check for empty values
    if (cashAmount > 1) {
      // set cash property to the amount inputted in the cash element
      portfolioObj["cash"] = Number(cashAmount);
    } else {
      // if there are empty values, error out
      errorMessage = true;
    }
    if (errorMessage == true) {
      return false; 
    }
    return true;
  };

  const calculateMetrics = async () => {
    updatePortfolio();

    const generateDate = () => {
      // current date in JS Date object
      let currentDate = new Date();
      // temporary date for weekend detection
      let tempDate = currentDate;
      // if the tempDate is on a Saturday or Sunday, minus a day from the currentDate object
      while (tempDate.getDay() === 6 || tempDate.getDay() === 0) {
        tempDate.setDate(currentDate.getDate() - 1);
      }
      let date = "";
      // get year
      date += tempDate.getFullYear();
      date += "-";
      // this ensures that if the digit is less than 10, it adds a 0 in front (Eg. - 04, instead of 5)
      date += ("0" + (tempDate.getMonth() + 1)).slice(-2);
      date += "-";
      date += ("0" + tempDate.getDate()).slice(-2);
      // example date - "2022-01-06";
      return date;
    };
    let DATE = generateDate();
    DATE = `2022-12-30`;
    console.log(DATE);
    if (Object.keys(portfolioObj["stocks"]).length >= 1) {
      for (const [key, value] of Object.entries(portfolioObj["stocks"])) {
        // When a user inputs a ticker,
        const TICKER = key;
        const AMOUNT = Number(value);
        const PRICE_URI = `https://www.alphavantage.co/query?function=TIME_SERIES_DAILY_ADJUSTED&symbol=${TICKER}&apikey=${API_KEY}`;
        const INFO_URI = `https://www.alphavantage.co/query?function=OVERVIEW&symbol=${TICKER}&apikey=${API_KEY}`;
        const INFO_DATA = await retreiveData(TICKER, "INFO", INFO_URI);
        const PRICE_DATA = await retreiveData(TICKER, "PRICE", PRICE_URI);
        console.log(PRICE_DATA);
        let stockPrice = 0;
        /**
         * Making sure the inputted ticker is good
         */
        // if the info_data response object keys is greater than one, and the "note" attribute (only given if there is an error)
        // is undefined, proceed - else, throw error. 
        if (
          Object.keys(INFO_DATA).length >= 1 &&
          INFO_DATA["note"] === undefined
        ) {

          stockPrice = Number(
            await PRICE_DATA["Time Series (Daily)"][`${DATE}`]["4. close"]
          );
          stockObjGraph[key] = Number((stockPrice * AMOUNT).toFixed(1));
          stockMetrics[TICKER] = {};
          stockMetrics[TICKER]["Beta"] = INFO_DATA["Beta"];
          stockMetrics[TICKER]["EPS"] = INFO_DATA["EPS"];
          stockMetrics[TICKER]["Dividend"] = INFO_DATA["DividendYield"];
          localStorage.setItem("stockObjGraph", JSON.stringify(stockObjGraph));
          localStorage.setItem("stockMetrics", JSON.stringify(stockMetrics));
          localStorage.setItem("cashAmount", String(cashAmount));
          errorMessage = false;
        } else {

          /**
           * Alert the user
           */
          errorMessage = true;
          deleteStock(TICKER);
          return false;
        }
      }
    }
    if (stockAmount < 1 || stockName.length < 1) {
      errorMessage = true; 
    }
    clearInputs();
    updateChart();
  };
  const updateChart = async () => {
    if (loaded) {
      let labels = [];
      let data = [];

      // console.log(Object.keys(stockObjGraph));
      console.log(stockObjGraph);
      console.log(portfolioObj);
      if (stockObjGraph == undefined) {
        stockObjGraph = {};
      }
      if (cashAmount > 1) {
        labels = ["Cash", ...labels];
        data = [cashAmount, ...data];
      }
      // console.log(stockObjGraph);
      if (Object.keys(portfolioObj["stocks"]).length < 1 && cashAmount < 1) {
        console.log("hello");
        labels = ["Blank"];
        data = [100];
      }
      // stockObjGraph["Cash"] = cashAmount;
      for (const [key, value] of Object.entries(stockObjGraph)) {
        console.log(key, value);
        labels = [key, ...labels];
        data = [value, ...data];
      }
      calculateAverages(data);
      console.log(stockObjGraph);
      console.log(labels);
      const parentElement = document.getElementById("graph");
      parentElement.innerHTML = "";
      parentElement.innerHTML = '<canvas size=400 id="doughnutChart" />';
      const childElement = document.getElementById("doughnutChart");

      // @ts-ignore

      // labels are all the stock names, plus cash if it is inputted
      // data is all the stock amount multiplied by latest price 
      chart = new Chart(childElement, {
        type: "doughnut",
        data: {
          // all stock names + cash
          labels: labels,
          datasets: [
            {
              label: "Portfolio",
              // stock values
              data: data,
              backgroundColor: [
                "rgb(255, 99, 132)",
                "rgb(54, 162, 235)",
                "rgb(255, 205, 86)",
              ],
              hoverOffset: 4,
            },
          ],
        },
        options : {
          maintainAspectRatio: false,
        }
      });
      errorMessage = false;
      return true;
    }
  };
  let averageEPS = 0;
  let averageBeta = 0;
  let averageDiv = 0;
  const calculateAverages = (data) => {
    // reduce the data down into its total value
    let portfolioTotalValue = data.reduce((/** @type {number} */ total, /** @type {number} */ next) => total + next, 0);
    // set initial variables for calculation
    let totalDivIncome = 0;
    let totalBeta = 0;
    let totalEPS = 0;
    // loop through the stockMetrics object, which is tied to the stockObject
    for (const [key, value] of Object.entries(stockMetrics)) {
      // total div income + the value of the looped stock multiplied by annual dividend
      totalDivIncome = totalDivIncome + stockObjGraph[key] * value["Dividend"];
      // total beta + the value of the looped stock multiplied by annual beta
      totalBeta = totalBeta + stockObjGraph[key] * value["Beta"];
      // total eps  + the value of the looped stock multiplied by annual eps
      totalEPS = totalEPS + stockObjGraph[key] * value["EPS"];
    };
    // to get average, divide the total income by the portfolio value
    averageDiv = Number(totalDivIncome / portfolioTotalValue);
    averageBeta = Number(totalBeta / portfolioTotalValue);
    averageEPS = Number(totalEPS / portfolioTotalValue);
  };
  const retreiveData = async (  
    /** @type {string} */ ticker,
    /** @type {string} */ mode,
    /** @type {RequestInfo | URL} */ URI
  ) => {
    // creates a custom key for localStorage
    // ticker is Stock ticker, and mode is whether the data is price data or stock overview data
    const KEY = `${ticker}-${mode}`;
    // sets portfolioObject key in localStorage
    localStorage.setItem("portfolioObject", JSON.stringify(portfolioObj));
    // if the object is stored, return it using the key
    if (localStorage.getItem(KEY) !== null) {
      return JSON.parse(localStorage.getItem(KEY));
    }
    // if it's not, fetch it
    else {
      const DATA = await (await fetch(URI)).json();
      // set the stock item with key
      localStorage.setItem(KEY, JSON.stringify(DATA));
      return DATA;
    }
  };
  const deleteStock = (/** @type {string} */ TICKER) => {
    let tempObj = stockObjGraph;
    let tempPortfolioObj = portfolioObj;
    let tempMetrics = stockMetrics;
    delete tempObj[TICKER];
    delete tempMetrics[TICKER];
    delete tempPortfolioObj["stocks"][TICKER];
    stockMetrics = tempMetrics;
    stockObjGraph = tempObj;
    portfolioObj = tempPortfolioObj;
    console.log(portfolioObj);

    // delete portfolioObj["stocks"][TICKER];
    updateChart();
  };
  const deleteCash = () => {
    cashAmount = 0;
    updateChart();
  };

  const clearInputs = () => {
    stockName = "";
    stockAmount = 0;
  };
</script>

<svelte:head>
  <link
    href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0-alpha1/dist/css/bootstrap.min.css"
    rel="stylesheet"
  />

  <link
    rel="stylesheet"
    href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.2.1/css/all.min.css"
  />
</svelte:head>
<!-- Dashboard -->

<nav class="navbar bg-dark sticky-top p-2">
  <div class="container-fluid">
    <a class="navbar-brand text-white" href="#">
      <div class="d-flex align-items-center justify-content-center">
        <i class="fa-solid fa-lg fa-money-bill-trend-up" />

        <span class="fs-2 ps-2">Stock Visualization</span>
      </div>

    </a>
  </div>
</nav>

<div class="container-fluid bg-light min-vh-100 p-0">
  <div class="row flex-nowrap">
    <div class="col-3 bg-secondary min-vh-100">
<div class="d-flex flex-column">
    
    <div class="d-flex align-items-center justify-content-center text-white">
      <i class="fa-solid fa-lg fa-shop" />
      <span class="fs-2 ps-2">Visualization</span>
    </div>

    <div class="d-flex align-items-center justify-content-center text-white">
      <i class="fa-solid fa-lg fa-shop" />
      <span class="fs-2 ps-2">Table</span>
    </div>

    <div class="d-flex align-items-center justify-content-center text-white">
      <i class="fa-solid fa-lg fa-shop" />
      <span class="fs-2 ps-2">Averages</span>
    </div>

    <div class="d-flex align-items-center justify-content-center text-white">
      <i class="fa-solid fa-lg fa-shop" />
      <span class="fs-2 ps-2 text-white">Add to Portfolio</span>
    </div>
</div>
      <!-- <div class="row">
        <div class="col-6 p-0">
          <div class="d-flex flex-column align-items-center">
            <h5>Stock visualizer and portfolio metric generation</h5>
            <div class="d-inline-flex p-1 align-items-center text-white">
              <i class="fa-solid fa-lg fa-shop" />
              
            </div>
            <div class="d-inline-flex p-1 align-items-center text-white">
              <i class="fa-solid fa-lg fa-shop" />
              <h3 class="m-0 p-2">Visualization</h3>
            </div>
            <div class="d-inline-flex p-1 align-items-center text-white">
              <i class="fa-solid fa-lg fa-shop" />
              <h3 class="m-0 p-2">Table</h3>
            </div>
            <div class="d-inline-flex p-1 align-items-center text-white">
              <i class="fa-solid fa-lg fa-shop" />
              <h3 class="m-0 p-2">Averages</h3>
            </div>
            <div class="d-inline-flex p-1 align-items-center text-white">
              <i class="fa-solid fa-lg fa-shop" />
              <h3 class="m-0 p-2">Add</h3>
            </div>
    
          </div>
        </div>
        <div class="col-6">
          <div class="d-flex flex-column align-items-center">
            <h3 class="m-0 p-2">Performance</h3>
          </div>
          
        </div>
      </div> -->

    </div>
    <div class="col-9">


      <div class="row px-5 py-3">
        <div class="col-sm-12">
          <div class="card">
            <div class="card-header">
              <!-- <div class="card-title"> -->
              <h5>Portfolio Visualization</h5>

              <!-- </div> -->
              <div class="card-body">
                <div class="" id="graph" />
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="row px-5 py-3">
        <div class="col-sm-12">
          <div class="card">
            <div class="card-header">
              <!-- <div class="card-title"> -->
              <h5>Portfolio Table</h5>

              <!-- </div> -->
              <div class="card-body">
                <table class="table">
                  <thead>
                    <tr>
                      <th scope="col">Stock</th>
                      <th scope="col">Quantity</th>
                      <th scope="col">Modify</th>
                    </tr>
                  </thead>
                  <tbody>
                    {#each Object.entries(portfolioObj["stocks"]) as [key, value]}
                      <tr>
                        <th scope="row">{key}</th>
                        <td>{value}</td>
                        <td
                          ><button
                            type="button"
                            class="btn btn-dark"
                            on:click={() => deleteStock(key)}>Delete</button
                          ></td
                        >
                      </tr>
                    {/each}
                    {#if cashAmount > 1}
                      <tr>
                        <th scope="row">Cash</th>
                        <td>{cashAmount}</td>
                        <td
                          ><button
                            type="button"
                            class="btn btn-dark"
                            on:click={() => deleteCash()}>Delete</button
                          ></td
                        >
                      </tr>
                    {/if}
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="row px-5 py-3">
        <div class="col-sm-12">
          <div class="card">
            <div class="card-header">
              <!-- <div class="card-title"> -->
              <h5>Portfolio Averages</h5>

              <!-- </div> -->
              <div class="card-body">
                <div class="d-flex justify-content-around">
                  <div class="card bg-transparent text-white">
                    <div class="card-header text-black">Average EPS</div>
                    <div class="card-body">
                      <div
                        class="d-inline-flex p-1 align-items-center text-black"
                      >
                        <i class="fa-solid fa-money-bills fa-2x" />
                        <h2 class="m-0 p-2">
                          {Math.round(averageEPS * 1000) / 1000}
                        </h2>
                      </div>
                    </div>
                  </div>
                  <div class="card bg-transparent text-white">
                    <div class="card-header text-black">
                      Average Annual Div Yield
                    </div>
                    <div class="card-body">
                      <div
                        class="d-inline-flex p-1 align-items-center text-black"
                      >
                        <i class="fa-solid fa-vault fa-2x" />
                        <h2 class="m-0 p-2">
                          {Math.round(averageDiv * 1000) / 1000}
                        </h2>
                      </div>
                    </div>
                  </div>
                  <div class="card bg-transparent text-white">
                    <div class="card-header text-black">Average Beta</div>
                    <div class="card-body">
                      <div
                        class="d-inline-flex p-1 align-items-center text-black"
                      >
                        <i class="fa-solid fa-arrow-up-right-dots fa-2x" />
                        <h2 class="m-0 p-2">
                          {Math.round(averageBeta * 1000) / 1000}
                        </h2>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="row px-5 py-3">
        <div class="col-sm-12">
          <div class="card">
            <div class="card-header">
              <!-- <div class="card-title"> -->
              <h5>Add to Portfolio</h5>

              {#if errorMessage}
                <div class="alert alert-danger" role="alert">
                  Please input valid information
                </div>
              {/if}
              <div class="d-flex flex-column justify-content-between">
                <div class="hstack gap-4 py-2">
                  <h4 class="p-0 m-0">Stock</h4>
                  <input
                    bind:value={stockName}
                    class="form-control me-auto"
                    type="text"
                    placeholder="Stock Ticker"
                    aria-label="Add your stock name here..."
                  />
                  <input
                    bind:value={stockAmount}
                    class="form-control me-auto"
                    type="number"
                    placeholder="Stock Amount"
                    aria-label="Add your stock amount here..."
                  />
                  <button
                    type="button"
                    class="btn btn-secondary"
                    on:click={() => calculateMetrics()}>Submit</button
                  >
                  <div class="vr" />
                  <button type="button" class="btn btn-outline-danger"
                    >Reset</button
                  >
                </div>
                <div class="hstack gap-3 py-2">
                  <h4 class="p-0 m-0">Cash</h4>
                  <input
                    bind:value={cashAmount}
                    class="form-control me-auto"
                    type="text"
                    placeholder="Stock Ticker"
                    aria-label="Add your stock name here..."
                  />
                  <!-- <input bind:value={stockName} class="form-control me-auto" type="number" placeholder="Stock Amount" aria-label="Add your stock amount here..."> -->
                  <button
                    type="button"
                    class="btn btn-secondary"
                    on:click={() => calculateMetrics()}>Submit</button
                  >
                  <div class="vr" />
                  <button type="button" class="btn btn-outline-danger"
                    >Reset</button
                  >
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
