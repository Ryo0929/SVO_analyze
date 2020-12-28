# SVO_analyze
從中文句子中抽取人物(主詞)，動作，受詞 

# Environments
使用百度DDParser 作為句法分析工具，請先參照官方指令安裝 [DDParser](https://github.com/baidu/DDParser)

# Usage
將中文句子中取 人物，動作，受詞抽取出來，並輸出成字典格式。如有動詞同位語，或多個主詞同樣也會被抽出。
```
text="他撥弄她的頭髮，時而捲動，時而打成一個結。"
syntax_analyze(text)
```
output: 
```
{'他': [('撥弄', '頭髮'), ('捲動', ''), ('打成', '結')]}
```
# Steps

### step1 將百度句法分析結果轉換成以單詞位置為key的字典，方便後面處理
```
result=ddp.parse(string)[0]
syntax_dict=dict(zip([x+1 for x in range(len(result["word"]))],list(zip(result["word"],result["head"],result["deprel"],result["postag"]))))
```
此時資料會長得像這樣
```
   {1: ('他', 2, 'SBV', 'r'),
    2: ('撥弄', 0, 'HED', 'v'),
    3: ('她', 5, 'ATT', 'r'),
    4: ('的', 3, 'MT', 'u'),
    5: ('頭髮', 2, 'VOB', 'n'),
    6: ('，', 2, 'MT', 'w'),
    7: ('時而', 8, 'ADV', 'd'),
    8: ('捲動', 2, 'COO', 'v'),
    9: ('，', 8, 'MT', 'w'),
    10: ('時而', 11, 'ADV', 'd'),
    11: ('打成', 2, 'COO', 'v'),
    12: ('一個', 13, 'ATT', 'm'),
    13: ('結', 11, 'VOB', 'n'),
    14: ('。', 11, 'MT', 'w')}
```

### step2 找人物(主詞)
找出句法結果字典中找出主詞也就是 deprel要等於"SBV"，因為我們要找的是人物所以詞性postag
```
s_index_list=[key for key,value in syntax_dict.items() if value[2]=="SBV" and value[3] in ["PER","r","ORG"]]
```
此時s_index_list會含有句子中主詞的座標
```
[1]
```
### step3 找動詞及動詞同位語
首先找出step1的主詞所指向的詞彙，如果該詞為動詞，則加入v_index_list。接著找出該棟詞是否有動詞同位語 "VV"或"COO" 指向該動詞。
```
s_v_dict={}
for s_index in s_index_list:
    #get index pointed by subject 
    v_index=syntax_dict[s_index][1]
    v_index_list=[]

    #add word to verb list if the pointed index is verb
    if syntax_dict[v_index][3]=="v":
        v_index_list=[v_index]

    #add word to verb list if verb have other verb appositive(同位語動詞)
    v_index_list=v_index_list+[key for key,value in syntax_dict.items() if value[2] in ["VV","COO"] and value[1] == v_index and value[3]=="v" ]

    # verb_index_list could be zero because subject's head cound be non verb and have no "VV" or "COO"
    if len(v_index_list)>0:
        s_v_dict.update({s_index:v_index_list})
 ```
 此時的s_v_dict應該會長得像這樣
 ```
 {1: [2, 8, 11]}
 ```
 ### step4 找動詞對應到的受詞
 找出詞語分析結果中，是否有包含受詞"VOB"且該詞指向(head)我們剛才找到的動詞。
 ```
    s_v_o_dict={}
    for s_index,v_list in s_v_dict.items():
        verb_object_list=[]
        for verb_index in v_list:
            object_index=None

            #find any object point to the verb
            for syn_index,syn_value in syntax_dict.items():
                # "VOB" is object, syn_value[1] is head index, syn_value[2] is part of speech
                if syn_value[1]==verb_index and syn_value[2]=="VOB":
                    object_index=syn_index
            verb_object_list=verb_object_list+[(verb_index,object_index)]
        s_v_o_dict.update({s_index:verb_object_list})
 ```
 此時的s_v_o_dict
 ```
 {1: [(2, 5), (8, None), (11, 13)]}
 ```
 
  ### step5 將座標轉換為詞語
```
result_text_dict={}
for key,value in s_v_o_dict.items():
    key_text=syntax_dict[key][0]
    v_o_tuple_list_text=[]
    for v_o_tuple in value:
        index_v=v_o_tuple[0]
        index_o=v_o_tuple[1]
        if index_o is not None:
            v_o_tuple_text=(syntax_dict[index_v][0],syntax_dict[index_o][0])
        else:
            v_o_tuple_text=(syntax_dict[index_v][0],"")
        v_o_tuple_list_text+=[v_o_tuple_text]
    result_text_dict.update({key_text:v_o_tuple_list_text})
``` 
result_text_dict
```
{'他': [('撥弄', '頭髮'), ('捲動', ''), ('打成', '結')]}
```
