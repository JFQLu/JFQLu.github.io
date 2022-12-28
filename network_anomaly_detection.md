##### Project Collaborators: *Andrew William, Dan Visitchaichan, Daniel Dinh, Michael Li*

### Project Details
This project involved collaborative work with 4 other data scientists over 10 weeks where we defined the problem, explored and transformed our data, researched solutions, implemented machine learning models to solve the probelm outlined and presented our deliverables to coordinators and other data science peers. The raw data was provided by CyAmast. There are many parts of this project which I have chosen to exclude from this blog due to length, however, I hope this will capture the key features of the project. 

### Background
The Internet of Things (IoT) is a technology that connects various devices, machines, and systems to a single network. It is widely used in many industries because it allows for efficient data gathering. However, these devices often have limited functions and are vulnerable to cyber attacks such as DDoS and MITM, making it important to have protection measures in place, such as network anomaly detection.

### Data
The data provided contained time series network data including packet/byte counts in/out of a number of ports of a number of devices. Below is a snapshot:
```python
data.info()
```
![image](https://user-images.githubusercontent.com/98208084/209839388-429df3b8-320f-4a0d-8de2-08be9d56f2d2.png)

**Here we hypothesise if we are able to classify a new datapoint into a particular flow we are able to observe when an anomaly occurs**

### Exploratory Data Analysis
Our team began by analysing and understanding the provided data. Python was used to calculate statistics and Matplotlib and seaborn packages were utilised to visualise the shape and trends of the time series data. Key observations include: 
- There was high correlation between corresponding in and out flows of each port
- There were distinct characteristics to each flow type (this is what we want for accurate classification)

### Feature Engineering
Our solution begins with the feature extraction step. This involved transforming raw training and testing data (provided by CyAmast) into training and testing feature datasets for subsequence sizes of 16, 32, 64, and 128. Note, we define a subsequence as a sequence of contiguous data from the original dataset which is transformed into a single instance in the feature datasets. The feature dataset contains 8 features for each labelled flow subsequence: average packet and byte counts for 1-, 2-, 4-, and 8-minute time frames. Furthermore, all observations which have only 0s in their 8 features were dropped from the dataset. 

A key function used in this process was the smoothen() function which allowed us to create higher time-frame data for extracting features:
```python
def smoothen(data,subsequence_length = 16,n=2):
    if len(data.shape) != 3:
      print("Subsequence is of wrong shape for our purposes, should be 3 dimensional --> (n_observations, subsequence size, n_original_features).") 

    # Makes an instance of a flow more coarse by n times.
    #print(f"length of flow instance:{len(subsequence)}")
    temp = np.split(data, subsequence_length/n,axis=1)
    temp = np.array(temp).sum(axis=-2)
    return temp
```

### Model Training
For this project we wanted to compare the effectiveness and logistics of both single-class and multi-class machine learning models in classifying network flows. For this reason we tackled this probelm with both single- and multi-class models. 

|Single-class| multi-class|
| ---------- | ---------- |
|GMM, K-Means, Fuzzy C-Means|Random Forest, XGBoost, AdaBoost, SVM, MLP, Logistic Regression| 

This blog will deep dive into the one-class Gaussian Mxture Model and the multi-class Random Forest models.

### Guassian Mixture Model - Deep Dive
A Gaussian mixture model (GMM) is a unsupervised probabilistic model that assumes that the data is generated from a mixture of several different Gaussian distributions. It is often used for classification tasks because it allows for the modeling of complex, multimodal distributions and can handle data with uncertainty or incomplete information.

In a GMM, each data point is assumed to belong to one of the Gaussian distributions in the mixture, and the model estimates the probability that a given data point belongs to each of the different distributions. The model can then classify a new data point based on which distribution it is most likely to belong to.

Once the GMM has been trained, it can be used to classify new data points by determining the distribution that they are most likely to belong to.

#### Preprocessing 
First any preprocessing is done including Z-score scaling and/or PCA; note that we perform experiments to determine if scaling or PCA improves model perforance.

#### Compute Optimal Cluster Number
The elbow method is a technique that is often used to determine the optimal number of clusters to use in a clustering algorithm such as a Gaussian mixture model (GMM). It is based on the idea that the optimal number of clusters is the point at which the decrease in the sum of squared distances between the points and their closest cluster centers starts to level off. 
```python
def compute_optimal_clusters(train,n_clusters,random_state):
    print("Calculating optimal number of clusters from elbow method")
    km = KMeans(random_state=random_state)
    visualizer = KElbowVisualizer(km, k=n_clusters,show=False)
    t1 = time.perf_counter()
    visualizer.fit(train)
    t2 = time.perf_counter()       
    optimal_k = visualizer.elbow_value_


    if optimal_k is None:
    optimal_k = 1

    print(f"Optimal number of clusters:{optimal_k}")
    optimal_k_time = t2-t1
    print('Time taken to find optimal clusters:',optimal_k_time,'s')

    return optimal_k,optimal_k_time
```
The above function returns us the optimal number of clusters *optimal_k*.

#### Training 
```python 
# Training a GMM using optimal number of clusters
gmm, training_time= fit_gmm(train,optimal_k,random_state = random_state,flow_type = flow_type)
```
The model is trained using sklearn's implementation of GMM. This model is then stored for testing. 

#### Testing 
Testing the GMM follows the following algorithm: 

**Input:**
Test feature data of same subsequence size as training feature data

Set of trained GMM classification models

Z-score scaler if *scale = True*

PCA scaler if *PCA = True*

**Output:**

Network flow predictions on test feature data

**for** each network flow **do**

>(1) **if** *scale = True* **then**
>>Scale each feature from test data using z-score scalar from training data; 

>(2) **if** *PCA = True* **then**
>> Obtain principal components using PCA scalar on features and reduce dimensions using minimal principal components;
 
>(3) Predict closest cluster to step 1 and 2 result, using all cluster models; 

>(4) Record distance between all closest clusters and step 1 and 2 result; 

**end**

(5) For each test data point, select cluster model with minimum distance predicted as winner model; 

(6) Use winner model's flow as predictions;

For single-class clasifiers like GMM we need to produce a model for each class. Predicting with single-class models such as GMM is more complicated then multi-class models due to the need for conflict resolution when the probablity measures are the same for more than one class (network flow). 

### Random Forest - Deep Dive
Random Forest (RF) is a supervised machine learning algorithm for classification and regression. It is an ensemble method meaning it combines the predictions of multiple decision trees trained on different subsets of the data, and makes a final prediction by taking the mode or mean of the individual predictions. It is effective at handling high-dimensional and sparse data, and is resistant to overfitting. It is also relatively easy to implement and interpret.

#### Preprocessing
Since RF is a supervised ML model we need to create labels in our feature data. This can be done by applying the following function to the data.

```python
def convert_one_class_to_multi_class(df):
    '''
    This function will convert the feature_subsequence dataset to make it suitable for multi-class.
    Note that columns that are not relevant to features (such as time/device_mac) will be removed.

    It will change the shape of the dataset from "wide" to "long", with the flow types assigned as a separate new column.

    NOTE: The input MUST be dataframe of features computed from subsequences.
    '''

    # Extract relevant column flow names, and feature names from the column names
    if 'time' in df.columns:
    df = df.drop(['time'],axis=1)

    if 'device_mac' in df.columns:
    df = df.drop(['device_mac'],axis=1)
    cols = [c for c in df.columns]
    flows = [extract_flow_from_column(c) for c in cols]
    features = [extract_feature_from_column(c) for c in cols]

    # Create a multi-level column with the extracted names
    new_columns = list(zip(flows,features,cols))
    df.columns=pd.MultiIndex.from_tuples(new_columns)

    # Stack the dataframe -
    # This will place the columns at the flow level, and original level into the rows. The end result will be our feature names will be the remaining columns
    df = df.stack(level= [0,2])
    # Condense the stacked dataframe by summing -
    # After stacking, there will be a lot of null values for observations that didn't exist. 
    # We can get rid of these observations using a group by and sum, since the nulls will be treated as 0 in a sum operation.
    df = df.groupby(level = [0,1]).sum()

    # Finally, reset index and rename accordingly
    df = df.reset_index().drop('level_0',axis=1).rename({'level_1':'FlowType'},axis=1)

    return df
```

### Training

Once we have labelled data we can simply train the RF model using sklearn's implementation. 

```python
rf = RandomForestClassifier(n_estimators=n_trees, random_state=random_state).fit(X_train,y_train)
```

### Testing 
Unlike GMM, only one model needs to be trained to classify all classes, no conflict resolution is required either. We only need one line to predict the class of any segment of network flow.
```python
y_pred = model.predict(X_test)
```

### Experiment Layer 1 Results
Best experiment results for each one-class model for layer 1
![image](https://user-images.githubusercontent.com/98208084/209847168-c7621fd3-1a31-4775-931c-2494d184905d.png)

Best experiment results for each multi-class model for layer 1
![image](https://user-images.githubusercontent.com/98208084/209847202-760f6db3-8842-40ef-af26-1c087d29bf66.png)

### Experiment Layer 2 Results
![image](https://user-images.githubusercontent.com/98208084/209859423-22bdf195-52f9-407c-92dd-6da2e2c245d7.png)

### Conclusion 
![image](https://user-images.githubusercontent.com/98208084/209859560-1a1232a9-bd95-4658-8825-48e6327a1b19.png)

#### Scalability Argument
To justify our overall final winning model, we need to briefly introduce the scalability argument. We take into account the advantage of one-class models like GMM over multi-class models where adding one new network flow would only require one new model to be trained, whilst the same scenario for multi-class models like random forest would require the entire model to be retrained over the new set of network flows.

However, we believe that the trade-off between time and performance is worthwhile.
Retraining the Random Forest would take ~10 minutes, and weighted average precision will be maintained at ~99%.
Alternatively, retraining GMM will take ~1 minute but weighted average precision will only be at the level about 87%.

Considering that the overall problem context is within the cybersecurity area, we should prioritise the weighted average precision metric over the slightly inferior training and testing times of Random Forest with respect to GMM.

Thus, the winning model of our classification task is **Random Forest*.

### Extending to Anomaly Detection
To integrate our ML solution into practive and extend its capabilities to anomaly detection we can simply observe if and when a particular flow's classification (or probability) deviates away from the actual flow in which the data is being collected from. 

### Best GMM and Random Forest (confusion matrix and classification report)
Guassian Mixture Models:

![image](https://user-images.githubusercontent.com/98208084/209861543-d8d78a92-0f39-4321-aeda-8619ea2595ca.png)
![image](https://user-images.githubusercontent.com/98208084/209861574-974e5549-adfd-4dca-a875-8f411bd7dcf2.png)

Random Forest:

![image](https://user-images.githubusercontent.com/98208084/209861212-4c290975-fe5f-4a5e-a715-d8a1b73872ba.png)
![image](https://user-images.githubusercontent.com/98208084/209861374-df34f367-be68-4132-8f42-9b7735437889.png)














