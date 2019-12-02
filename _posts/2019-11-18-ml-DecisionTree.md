---
layout: default
title: 机器学习笔记----决策树模型
categories: [机器学习]
---
<center><font size="14"> 决策树</font></center>
依然从这几个问题入手：   

1. 学什么？   
1. 怎么用？   
1. 怎么学？   
1. 小结
本次从Python著名的机器学习库[sklearn](https://sklearn.apachecn.org/)。
跟着[官网文档](https://scikit-learn.org/stable/modules/tree.html)从鸢尾花数据集开始。
<img src='/static/image/ml/iris-data.png' /> 
鸢尾花(iris)数据集分析 Iris 鸢尾花数据集是一个经典数据集，在统计学习和机器学习领域都经常被用作示例。数据集内包含 3 类共 150 条记录，每类各 50 个数据，每条记录都有 4 项特征：花萼长度、花萼宽度、花瓣长度、花瓣宽度，可以通过这4个特征预测鸢尾花卉属于（iris-setosa, iris-versicolour, iris-virginica）中的哪一品种。

### 学什么？
在决策树模型中，机器学习学到一棵决策树

<img src='/static/image/ml/iris.png' /> 

### 怎么用？
单看图就已经很清楚了，通过判断数据的每一个属性，通过一系列的if/else找到最终结果。
可以尝试用代码来表示一下：

``` c++
//预测结果就是遍历决策树
DNode*  predict(DNode* decisionTree,vector<float>vAtrr){
    DNode* ret = decisionTree;
    while (ret != NULL && ret->wIndex != -1){
        if (vAtrr[ret->wIndex] <=ret->wValue){//决策树的判断条件
            ret = ret->left;
        } else {
            ret = ret->right;
        }
    }
    return ret;
}
```
调用方法也很简单，将鸢尾花数据传入predict即可
``` c++
int main() {
    //通过解析sklearn保存的dot文件来生成一棵决策树
    DNode* decisionTree = ParseDecisionTree("/tmp/DecisionTree.dot");

    vector<vector<float>> X = { {5.1, 3.5, 1.4, 0.2},
                                  {4.9, 3.,  1.4, 0.2},
                                  ....};
    
    for (vector<vector<float>>::iterator it =  X.begin();
        it != X.end();
        ++it
    ){
        //获得预测结果
        DNode* ret = predict( decisionTree,*it);
        if (ret != NULL) {
            printf("%d,", ret->classIndex);
        } else {
            printf("error when predict");
        }
    }
    //最后记得释放decisionTree的内存。
     stack<DNode*> stackDnode;
    DNode* current = decisionTree;
    stackDnode.push(current);
    stackDnode.push(current); // 每个结点 push 两次，这样可以简单的判断出哪些结点是否处理过
    while (!stackDnode.empty()) {
        current = stackDnode.top();
        stackDnode.pop();
        if (!stackDnode.empty() && current == stackDnode.top()) {
            if (current->right != NULL) {
                stackDnode.push(current->right);
                stackDnode.push(current->right);
            }
            if (current->left != NULL) {
                stackDnode.push(current->left);
                stackDnode.push(current->left);
            }
        } else {
            delete current;
           // printf("%d,%f,%d\n",current->classIndex,current->wValue,current->wIndex);
        }
    }
}
```
为了完整起见，贴上决策树节点的定义。
``` c++
class DNode{
public:
    DNode(string attName,float wValue,string className){
        attName = trim(attName);
          if (attName.compare("sepal length (cm)") == 0){
            wIndex = 0;
        }else  if (attName.compare("sepal width (cm)") == 0){
            wIndex = 1;
        } else if (attName.compare("petal length (cm)") == 0){
            wIndex = 2;
        } else if (attName.compare("petal width (cm)") == 0){
            wIndex = 3;
        } else  if (attName.size() == 0){
              wIndex = -1;
        } else {
            printf("error!!!!,[%s]",attName.c_str());
        }
        left = NULL;
        right = NULL;
        this->wValue = wValue;

        className = trim(className);
        if (className.compare("setosa") == 0){
            classIndex = 0;
        } else  if (className.compare("versicolor") == 0){
            classIndex = 1;
        } else  if (className.compare("virginica") == 0){
            classIndex = 2;
        } else {
            classIndex = -1;
            printf("error,when parse classname!!!!,[%s]",className.c_str());
        }
    }
    int wIndex;//属性判断索引
    float wValue;//判断条件值
    int classIndex;
    DNode *left;
    DNode *right;
};
```
为了完整起见，贴上我自己写的dot文件解析过程：
```c++
int split(const string &str, vector<string> &ret_v, const string &sep) {
    if (str.empty()) {
        return 0;
    }
    string tmp;
    string::size_type pos_begin = str.find_first_not_of(sep);
    string::size_type comma_pos = 0;


    while (pos_begin != string::npos) {
        comma_pos = str.find(sep, pos_begin);
        if (comma_pos != string::npos) {
            tmp = str.substr(pos_begin, comma_pos - pos_begin);
            pos_begin = comma_pos + sep.length();

        } else {
            tmp = str.substr(pos_begin);
            pos_begin = comma_pos;
        }


        if (!tmp.empty()) {
            ret_v.push_back(tmp);
            tmp.clear();

        }
    }
    return 0;
}

//去除空格
string trim(const string& str)
{
    string::size_type pos = str.find_first_not_of(' ');
    if (pos == string::npos)
    {
        return str;
    }
    string::size_type pos2 = str.find_last_not_of(' ');
    if (pos2 != string::npos)
    {
        return str.substr(pos, pos2 - pos + 1);
    }
    return str.substr(pos);
}

DNode* ParseDecisionTree(string fn) {
    ifstream infile;
    string sline;
    infile.open("/Users/zhoujunsheng/tmp/DecisionTree.dot");
    int i = 0;
    map<string,DNode*> mapDnode;
    while (getline(infile, sline)) {
        if (i++<3){
            continue;
        }
        if (sline.compare("}") == 0){
            continue;
        }
        //printf("%s\n",sline.c_str());
        string::size_type pos_begin = sline.find_first_of(" ");
        string strIndex = sline.substr(0, pos_begin);
        strIndex = trim(strIndex);
        //获得节点索引
        //printf("index=%s\n",strIndex.c_str());
        //获得 label
        pos_begin = sline.find_first_of("<");
        if (std::string::npos != pos_begin){
            //printf("%s\n========\n",sline.c_str());
            pos_begin++;
            string::size_type pos_end = sline.find_last_of(">");
            string strLabel = sline.substr(pos_begin, pos_end-pos_begin);
            //printf("lable=%s\n",strLabel.c_str());

            //按<br/>换行
            vector<string> sVec;
            split(strLabel,sVec,"<br/>");
            for(vector<string>::iterator it = sVec.begin();it != sVec.end();++it){
                // printf("%s\n",it->c_str());
            }
            if (std::string::npos != sVec[0].find_last_of("&le;")) {
                //中间节点
                //第一行是判断条件
                vector<string> vjudge;
                split(sVec[0], vjudge, "&le;");
                //printf("%s<=%s\n",vjudge[0].c_str(),vjudge[1].c_str());
                int index = atoi(sVec[1].c_str());
                vector<string> vLable;
                split(sVec[1], vLable, " ");

                vector<string> vclass;
                split(sVec[4], vclass, "=");
                // printf("class:=%s\n",vclass[1].c_str());
                DNode *dnode = new DNode(vjudge[0], atof(vjudge[1].c_str()), vclass[1]);
                mapDnode[strIndex] = dnode;

            } else {
                //叶子节点
                vector<string> vclass;
                split(sVec[3], vclass, "=");
                // printf("class:=%s\n",vclass[1].c_str());
                DNode *dnode = new DNode("", 0, vclass[1]);
                //mapDnode.insert(std::make_pair(strIndex,dnode));
                mapDnode[strIndex] = dnode;
                //printf("mapDnode[%s]=%f\n",strIndex.c_str(),mapDnode[strIndex]->wValue);
            }
        } else {
            sline = sline.substr(0, sline.size() - 1);
               vector<string> vFT;
            split(sline, vFT, " ");
            string from = trim(vFT[0]);
            string to = trim(vFT[2]);
            if (mapDnode[from]->wIndex == -1){
                printf("error When parse link:%s\n",sline.c_str());
            }
            //printf("from mapDnode[%s]=%f\n",from.c_str(),mapDnode[from]->wValue);
            //printf("to mapDnode[%s]=%f\n",to.c_str(),mapDnode[to]->wValue);
            if (mapDnode[from]->left == NULL){
                mapDnode[from]->left = mapDnode[to];
            } else {
                if (mapDnode[from]->right == NULL){
                    mapDnode[from]->right = mapDnode[to];
                } else {
                    printf("error When parse link,overlap right:%s\n",sline.c_str());
                }
            }
        }
    }
    return mapDnode["0"];
}
```



### 怎么学？
在Python中使用Sklearn库，学习决策树是一件很容易的事
```python
from sklearn.datasets import load_iris
from sklearn import tree
iris = load_iris()#加载鸢尾花数据集
clf = tree.DecisionTreeClassifier()
clf = clf.fit(iris.data, iris.target)
#保存模型
tree.export_graphviz(clf, out_file="/tmp/DecisionTree.dot", 
                      class_names=iris.target_names,
                      filled=True, rounded=True,  
                     special_characters=True)
```

### 小结：
* 决策树通过询问一系列二元选择题来做预测。
* 若想生成决策树，就要不断拆分数据样本以获得同质组，直到满足终止条件。这个过程被称为递归拆分。
* 虽然决策树易于使用和理解，但是容易造成过拟合问题，导致出现不一致的结果。为了尽量避免出现这种情况，可以采用随机森林等替代方法。