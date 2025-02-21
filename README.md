```
 ___       _       _ _ _                  _     _____              _ _               ____        _ 
|_ _|_ __ | |_ ___| | (_) __ _  ___ _ __ | |_  |_   _| __ __ _  __| (_)_ __   __ _  | __ )  ___ | |_
 | || '_ \| __/ _ \ | | |/ _` |/ _ \ '_ \| __|   | || '__/ _` |/ _` | | '_ \ / _` | |  _ \ / _ \| __|
 | || | | | ||  __/ | | | (_| |  __/ | | | |_    | || | | (_| | (_| | | | | | (_| | | |_) | (_) | |_ 
|___|_| |_|\__\___|_|_|_|\__, |\___|_| |_|\__|   |_||_|  \__,_|\__,_|_|_| |_|\__, | |____/ \___/ \__|
                         |___/                                               |___/                   
₿   Ξ   ₳   ₮   ✕   ◎   ●   Ð   Ł   Ƀ   Ⱥ   ∞   ξ   ◈   ꜩ   ɱ   ε   ɨ   Ɓ   Μ   Đ  ⓩ  Ο   Ӿ   Ɍ  ȿ
```

> [![https://t.me/intelligent_trading_signals](https://img.shields.io/badge/Telegram-2CA5E0?logo=telegram&style=for-the-badge&logoColor=white)](https://t.me/intelligent_trading_signals) 📈 **<span style="font-size:1.5em;">[Intelligent Trading Signals](https://t.me/intelligent_trading_signals)</span>** 📉 **<https://t.me/intelligent_trading_signals>**

# Intelligent trading bot

The project is aimed at developing an intelligent trading bot for automatic trading cryptocurrencies using state-of-the-art machine learning approaches to data processing and analysis. The project provides the following major functions:
* Analyzing historic data and training machine learning models as well as finding their best hyper-parameters. It is performed in batch off-line mode.
* Signaling service which is regularly requests new data from the exchange and generates buy-sell signals by applying the previously trained models
* Trading service which does real trading by buying or selling the assets according to the generated signals

Note that is an experimental project aimed at studying how various machine learning and feature engineering methods can be applied to cryptocurrency trading. 

# Intelligent trading channel

This software is running in a cloud and sends its signals to this Telegram channel:

📈 **[Intelligent Trading Signals](https://t.me/intelligent_trading_signals)** 📉 **<https://t.me/intelligent_trading_signals>**

Everybody can subscribe to the channel to get the impression about the signals this bot generates.

Currently, the bot is configured using the following parameters:
* Exchange: Binance
* Cryptocurrency: ₿ Bitcoin
* Analysis frequency: 1 minute (currently the only option)
* Score between -1 and +1. <0 means likely to decrease, and >0 means likely to increase
* Filter: notifications are sent only if score is greater than ±0.20 (can be parameterized)
* One increase/decrease sign is added for each step of 0.05 (exceeding the filter threshold) 
* Prediction horizon 3 hours ahead. For example, if the score is +0.25 then the price is likely to increase 1-2% during next 3 hours. Note that the algorithm is trained for high (for increase) and low (for decrease) prices - not for close prices
* History taken into account for forecasts: 12-24 hours

There are silent periods when the score in lower than the threshold (currently 0.15) and no notifications are sent to the channel. If the score is greater than the threshold, then every minute a notification is sent which looks like 

> ₿ 60,518 📉📉📉 Score: -0.26

The first number is the latest close price. The score -0.26 means that it is very likely to see the price 1-2% lower than the current close price during next few hours. The three decrease signs mean three 0.5 steps after the threshold 0.15.

# Signaler service

Every minute, the signaler performs the following steps to make a prediction about whether the price is likely to increase or decrease:
* Retrieve the latest data from the server and update the current data window which includes some history
* Compute derived features based on the nearest history collected (which now includes the latest data)
* Apply several (previously trained) ML models by forecasting some future values (not necessarily prices) which are also treated as (more complex) derived features. We apply several forecasting models (currently, Gradient Boosting, Neural network, and Linear regression) to several target variables describing future decrease, future increase with different horizons
* Aggregate the results of forecasting produced by different ML models and compute the final score which reflects the strength of the upward or downward trend. Here we use many previously computed scores as inputs and derive one output score. Currently, it is implemented as an aggregation procedure but it could be based on a dedicated ML model trained on previously collected scores and the target variable. Positive score means growth and negative score means fall
* Use the final score for notifications

Notes:
* The final result of the signaler is the score (between -1 and +1). The score should be used for further decisions about buying or selling by taking into account other parameters and data sources.
* For the signaler service to work, trained models have to be available and stored in the model folder `model_folder`. The models are trained in batch mode and the process is described in the corresponding section.

Starting the service: `python3 -m service.server -c config.json`

# Training machine learning models

For the signaler service to work, a number of ML models must be trained and the model files available for the service. All scripts run in batch mode by loading some input data and storing some output files. The scripts are implemented in the `scripts` module.

The following batch scripts are used to train the models needed by the signaler:
* Download the latest historic data: `python -m scripts.download_data -c config.json`
  * The result is one output file with the name pattern: `{symbol}-{freq}-klines.csv`
  * It uses Binance API but you can use any other data source or download data manually using other scripts
* Merge several historic datasets into one dataset: `python -m scripts.merge_data -c config.json`
  * This script solves two problems: 1) there could be other sources like depth data or futures 2) a data source may have gaps so we need to produce a regular time raster in the output file
  * The result is one output file with the name pattern: `{symbol}-{freq}.csv`
* Generate feature matrix: `python -m scripts.generate_features -c config.json`
  * This script computes all derived features and labels defined programmatically
  * The result is one output file with the name pattern: `{symbol}-{freq}-features.csv`
  * It may take hours to compute because it runs in non-incremental mode, that is, computes the features for all the available data even if only the latest small portion was really updated
* Train prediction models: `python -m scripts.train_predict_models -c config.json`
  * This script uses all input features and all generated labels to train several ML models with pre-defined hyper-parameters
  * Hyper-parameter tuning is not part of this procedure, that is, it is assumed that we already have good hyper-parameters
  * The results are stored as multiple model files in the model folder and the file names encode the following model dimensions: label being used as a target variable like `high_10`, input data like `k` (klines) or `f` (futures), algorithm like `gb` (gradient boosting), `nn` (neural network) or `lc` (linear classifier).
  * The model folder also contains `metrics.txt` with the scores of the trained models

# Hyper-parameter tuning

There are two problems:
* How to choose best hyper-parameters for our ML models. This problem can be solved in a classical way, e.g., by grid search. For example, for Gradient Boosting, we train the model on the same data using different hyper-parameters and then select those showing best score. This approach has one drawback - we optimize it for best score which is not trading performance, which means that the trading performance is not guaranteed to be good (and in fact it will not be good). Yet, we do not have any other approach. As a workaround, we use this score as an intermediate feature with the goal to optimize trading performance on later stages.
* If we compute the final aggregated score (like +0.21), then the question is should we buy, sell or do nothing? In fact, it is the most difficult question. To help answer it, additional scripts were developed for backtesting and optimizing buy-sell signal generation:
  * Generate rolling predictions which simulate what we do by regularly re-training the models and using them for prediction: `python -m scripts.generate_rolling_predictions -c config.json`
  * Train signal models for choosing best thresholds for sell-buy signals which produce best performance on historic data: `python -m scripts.train_signal_models -c config.json` 

Yet, this advanced level of data analysis is work in progress, and it is also the most challenging part because here we cannot rely on the conventional ML approach. The goal is to find parameters to optimize the trading strategy which is a sequence of buy and sell transaction. Such a scenario is difficult to formally describe in conventional ML terms. The provided scripts are aimed at helping in such optimizations because they can generate data and then test different trading (currently, rule-based) strategies.

# Configuration parameters

The configuration parameters are specified in two files:
* `service.App.py` in the `config` field of the `App` class
* `-c config.jsom` argument to the services and scripts. The values from this config file will overwrite those in the `App.config` when this file is loaded into a script or service

Here are some most important fields (in both `App.py` and `config.json`):
* `symbol` it is a trading pair like `BTCUSDT` - it is important for almost all cases
* `data_folder` - location of data files which are needed only for batch scripts and not for services
* `model_folder` - location of trained ML models which are stored by batch scripts and then are loaded by the services
* Analyzer parameters. These mainly columns names.
  * `labels` List of column names which are treated as labels. If you define a new label used for training and then for prediction then you need to specify its name here. Note that we use multiple target variables (e.g., with different prediction horizons) and multiple prediction algorithms.
  * `class_labels_all` It is not used by the system and is created for convenience by listing *all* labels we compute so that it is easier to choose labels we want to experiment with during hyper-parameter tuning.
  * `features_kline` List of all column names used as input features for training and prediction.
  * `features_futur` Experimental. Currently, not used. Features based on future prices.
  * `features_depth` Experimental. Currently, not used. Features based on market depth (order book data).
* `signaler` is a section for signaler parameters
  * `notification_threshold` It is an integer like 0, 1, 3, 4 etc., which specifies the score threshold for sending notifications. Instead of using an absolute continuous threshold like 0.123, we specify the number of steps each step being (currently) equal 0.05. If you want to receive *all* notifications every minute, then set this parameter to 0. If you want to receive messages if score is greater than 0.10, then set it to 2. The notifier will also add one or more icons to each message and this number of icons is also equal to the number of 0.05 intervals exceeding the current threshold.
  * `analysis.features_horizon` This parameter specifies maximum history length used for training and prediction. The unit is the number of rows (not time). The system must know how much previous quotes is needed in order to be able to compute derived features and make predictions. For example, if we use rolling mean with window size 60 (1 hour in the case of minute data), then this parameter has to be equal 60. We suggest a higher value like 70 to guarantee that all 60 measurements are available. This parameter needs to be changed if you change your feature definitions, particularly, in `common.feature_generation.py`.
* `trader` is a section for trader parameters. Currently, not thoroughly tested.
* `collector` These parameter section is intended for data collection services. There are two types of data collection services: synchronous with regular requests to the data provider and asynchronous streaming service which subscribes to the data provider and gets notifications as soon as new data is available. They are working but not thoroughly tested and integrated into the main service. The current main usage pattern relies on manual batch data updates, feature generation and model training. One reason for having these data collection services is 1) to have faster updates 2) to have data not available in normal API like order book (there exist some features which use this data but they are not integrated into the main workflow).

Here is a sample `config.json` file:
```json
{
  "api_key": "<binance-key-only-for-trading>",
  "api_secret": "<binance-secret-only-for-trading>",

  "telegram_bot_token": "<source-chat-id>",
  "telegram_chat_id": "<destination-chat-id>",

  "symbol": "BTCUSDT",
  "base_asset": "BTC",
  "quote_asset": "USDT",

  "data_folder": "C:/DATA2/BITCOIN/GENERATED/BTCUSDT",
  "model_folder": "C:/DATA2/BITCOIN/MODELS/BTCUSDT"
}
```

# Trader

The trader is working but not thoroughly debugged, particularly, not tested for stability and reliability. Therefore, it should be considered a prototype with basic functionality. It is currently integrated with the Signaler but in a better design should be a separate service.
