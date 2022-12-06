## StockSage

This is an application for virtual trading of the stocks, maintaining a watchlist of stock. The applications Code base can be broadly classified as having the following parts :

1. Backened Codebase
    - Yahoo Finance API
    - Authentication Service using Firebase
    - Room Database for Storage
    - Caching Mechanism
2. Frontend Codebase
    - Recycler View for Stock List
    - Infinite animation banner
    - Graphing for charts of stocks
    - Different Activity views


I was responsible for the following parts of the application, below I will be presenting an enumerated list of my contribution for the project code wise and otherwise. Lets start with the **code contributions**

- Yahoo Finance API

    We required an API for fetching the stock information, while most library providing this service were paid I researched viability of the Yahoo Finance API for two task which were critical for our application

        1. Fetching the real time stock price 
        2. Ability to search a stock using its name

    To this end the following classes were created in a package:
    ```kotlin
    class YahooFinanceQuoteProvider{

    private val baseUrl = "https://query1.finance.yahoo.com/v7/finance/quote?symbols="
    private val userAgent = "Mozilla/5.0 (Linux; Android) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Mobile Safari/537.36"

    fun getStockQuote(symbol: String): JSONObject? {
        val endpoint = URL(baseUrl + URLEncoder.encode(symbol, "utf-8"))
        val connection: HttpsURLConnection = endpoint.openConnection() as HttpsURLConnection
        connection.setRequestProperty("User-Agent", userAgent);

        if (connection.responseCode == 200) {
            connection.inputStream.use { inputStream ->
                val jsonString = inputStreamToString(inputStream)
                val rootObject = JSONObject(jsonString)
                val quoteResponseObject = rootObject.getJSONObject("quoteResponse")
                val resultObject = quoteResponseObject.getJSONArray("result").getJSONObject(0)
                return resultObject
            }

        } else
            throw Exception("Error: YahooFinanceQuoteProvider response code " + connection.responseCode)
    }
    ```

    ```kotlin
    class YahooFinanceSearch {
    private val SEARCH_URL = "https://query1.finance.yahoo.com/v1/finance/search?q="
    private val USER_AGENT = "Mozilla/5.0 (Linux; Android) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Mobile Safari/537.36"

    fun search(symbol: String): List<StockData> {
        val endpoint = URL(SEARCH_URL + URLEncoder.encode(symbol, "utf-8"))
        val connection: HttpsURLConnection = endpoint.openConnection() as HttpsURLConnection
        connection.setRequestProperty("User-Agent", USER_AGENT)

        if (connection.responseCode == 200) {
            connection.inputStream.use { inputStream ->
                val jsonString = inputStreamToString(inputStream)
                val rootObject = JSONObject(jsonString)
                return convertJsonToStocksYahoo(rootObject.getJSONArray("quotes"))
            }
        } else {
            throw Exception("Error: YahooFinanceSearch response code " + connection.responseCode)
        }
    }
    }
    ```
    These fecthed values required required a cache to maintained so we donot fetch values each time when we are using the application, instead fetch value only if the new value was not fetched within the last 1 minute. This allowed us to restrict the number of API calls were making, as we had restriction on the number of calls we can make to the FREE tier of the Yahoo finance API

    ```kotlin
    object StockDataCache {

    var stockData : MutableMap<String, StockItem> = mutableMapOf()
    var stockPorfolioData : MutableMap<String, Double> = mutableMapOf()

    fun getStocks() : List<StockItem>
    {
        val stockList : MutableList<StockItem> = mutableListOf()
        stockData.forEach{entry ->
            stockList.add(entry.value)
        }
        return stockList
    }

    fun setStocks(stocks : MutableList<StockItem>)
    {
        // clear the old cache and update
        stockData.clear()
        stocks.forEach{
            entry ->
            stockData[entry.symbol] = entry
        }
    }

    fun addStockToCache(stock : StockItem)
    {
        ...
    }

    private fun checkIfPresentInStockData(queryStock : StockItem) : Boolean
    {
        ...
    }

    private fun checkIfPresentInStockPortfolioData(queryStock : String) : Boolean
    {
        ...
    }

    }
    ```

- Infinite banner

    We wanted to show some informative images to improve user expierence while the use was registering in the application, to this, I used an Recycle view combined with animation to develop a infinite banner for the registration page. 

    This used OOPS concept to make it resuable by diffferent application. The below abstract class was created which can be implemented in you own class shown below to add infinite animation horizontal recycle view 
    ```java
    public abstract class AbsBannerAdapter {
    private final DataSetObservable mDataSetObservable = new DataSetObservable();

    void registerDataSetObserver(DataSetObserver observer) {
        mDataSetObservable.registerObserver(observer);
    }

    void unregisterDataSetObserver(DataSetObserver observer) {
        mDataSetObservable.unregisterObserver(observer);
    }

    public void notifyDataSetChanged() {
        mDataSetObservable.notifyChanged();
    }

    public void notifyDataSetInvalidated() {
        mDataSetObservable.notifyInvalidated();
    }

    protected abstract int getCount();

    protected abstract View makeView(InfiniteBannerView parent);

    protected abstract void bind(View view, int position);
    }
    ```

    ```java 
    public class Banner extends AbsBannerAdapter {
    @Override
    public int getCount() {
        return 4;
    }

    @Override
    protected View makeView(InfiniteBannerView parent) {
        ImageView imageView = new ImageView(parent.getContext());
        imageView.setLayoutParams(new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
        imageView.setScaleType(ImageView.ScaleType.FIT_CENTER);
        return imageView;
    }

    @Override
    protected void bind(View view, int position) {
        if (position == 0) {
            ((ImageView) view).setImageResource(R.drawable.img_0);
        } else if (position == 1) {
            ((ImageView) view).setImageResource(R.drawable.img_1);
        } else if (position == 2){
            ((ImageView) view).setImageResource(R.drawable.img_2);
        }
        else {
            ((ImageView) view).setImageResource(R.drawable.img_3);
        }
    }
    }
    ```

- Room Database

    We required data to persist between sessions, as well maintain transactions of buying/ selling of stock within the application, room database came to out rescue, where I created the database as well the corresponding API to insert/ delete/ get transactions from the database.

    **Database Connection class**, This uses `Flow` from kotlin when we need to fetch data asynchronously in the background, it also has syncrohonous version of the same API for inline fetching.
    ```kotlin
    @Dao
    interface PortfolioDataDao {

        @Query("SELECT * FROM stock")
        fun getAll() : Flow<List<PortfolioDataEntity>>

        @Query("SELECT * FROM stock")
        fun getAllSync() : List<PortfolioDataEntity>

        @Query("SELECT * FROM stock WHERE name LIKE :queryName")
        fun findByName(queryName : String) : Flow<PortfolioDataEntity>

        @Query("SELECT * FROM stock WHERE symbol = :querySymbol")
        fun findBySymbol(querySymbol : String) : LiveData<List<PortfolioDataEntity>>

        @Query("SELECT * FROM stock WHERE symbol = :querySymbol")
        fun findBySymbolSync(querySymbol : String) : List<PortfolioDataEntity>

        @Insert(onConflict = OnConflictStrategy.REPLACE)
        fun insertALL(vararg stocks: PortfolioDataEntity)

        @Insert(onConflict = OnConflictStrategy.REPLACE)
        fun insertOne(stock: PortfolioDataEntity)

        @Delete
        fun delete(stock: PortfolioDataEntity)
    }

    ```

    **Database class** This returns the syncrohonised version of the class so we only create one instance of the database per instance of the application.
    ```kotlin
        @Database(entities = [PortfolioDataEntity::class], version = 1)
    abstract class PortfolioDatabase : RoomDatabase() {
        abstract fun getDao() : PortfolioDataDao

        companion object {
            @Volatile
            private var INSTANCE: PortfolioDatabase? = null

            fun getDatabase(context: Context) = INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    PortfolioDatabase::class.java,
                    "forageable_database"
                ).build().apply {
                    INSTANCE = this
                }
            }

        }
    ```
- Github 

    I was also responsible for mainting the version control for out application, we developed different features in different branch 