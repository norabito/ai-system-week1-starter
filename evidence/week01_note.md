# Week 1 Evidence
學號：7114029034
姓名：范睿凱

1. Dataset 在哪裡？

ANS: 'data/customer_intent_demo.csv'



2. Baseline 在哪裡？

ANS: 'src/rule_baseline.py'



3. 本次 Accuracy = ___0.950___



4. 找一個 Failure Case：輸入／正確答案／Baseline 預測

ANS: 輸入:商品下單後配送需要冷藏嗎/intent: product_info/prediction: order_delivery



5. 為什麼 Baseline 會錯？（一句話）

ANS: 語句包含下單、配送等詞，可能導致模型判斷失準，無法理解真正意圖



6. 這是否代表現在一定要使用 AI？為什麼？（一句話）

ANS: 當前結果準確率高，可以先針對錯誤判斷的部分對模型做微調，後續若資料或語意更加複雜再考慮引入AI即可