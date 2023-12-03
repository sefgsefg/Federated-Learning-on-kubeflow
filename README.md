# FL-on-kubeflow
A simple example of performing federated learning on kubeflow

Part of the code comes from "https://github.com/stijani/tutorial?fbclid=IwAR2AvmE3DzXzuF6MxHuVUaP7_KLyOVIZK679d548jR2Gx4PlXKjZOU_DzuM"

## Description
Installation
---
```
pip install kfp
```
Server
---
We use the "Flask" to make a server.
First create the list for storing client's upload data and make a dictionary for sharing variable between clients context

You should change "NUM_OF_CLIENTS" number for actual number of clients. This example is 2.
```
app = Flask(__name__)
    clients_local_count = []
    scaled_local_weight_list = []
    global_value = { #Share variable
                    'last_run_statue' : False, #last run finish or not
                    'data_statue' : None,      
                    'global_count' : None,     #global_count finish or not
                    'step' : None,
                    'weight_statue' : None,
                    'average_weights' : None,
                    'shutdown' : 0}
    
    NUM_OF_CLIENTS = 2 #number of clients
```
Create locks to ensure that the server calculation is correct
```
init_lock = threading.Lock()
clients_local_count_lock = threading.Lock()
scaled_local_weight_list_lock = threading.Lock()
cal_weight_lock = threading.Lock()
shutdown_lock = threading.Lock()
```
In the begining, the first enter client will lock and init the global variable, subsequent clients do not need to do it again.

Then the server will get client's data
```
@app.route('/data', methods=['POST'])
    def flask_server():
        with init_lock:  #check last run is finish and init variable
            
            while True:
                
                if(len(clients_local_count)==0 and global_value['last_run_statue'] == False):#init the variable by first client enter
                    global_value['last_run_statue'] = True
                    global_value['data_statue'] = False
                    global_value['scale_statue'] = False
                    global_value['weight_statue'] = False
                    break
                
                elif(global_value['last_run_statue'] == True):
                    break
                time.sleep(3)
        
        local_count = int(request.form.get('local_count'))          #get data
        bs = int(request.form.get('bs'))
        local_weight = json.loads(request.form.get('local_weight'))
        local_weight = [np.array(lst) for lst in local_weight]
```
Here is example for locking process.  The first enter client will detect the length of "clients_local_count" equal to NUM_OF_CLIENTS,
and set "global_value['data_statue'] " to True.  

The subsequent clients will enter "elif" part to prevent doing the same action.
```
with clients_local_count_lock:
            clients_local_count.append(int(local_count))
            
with scaled_local_weight_list_lock:
    while True:
        
        if (len(clients_local_count) == NUM_OF_CLIENTS and global_value['data_statue'] != True):
            global_value['last_run_statue'] = False
            sum_of_local_count=sum(clients_local_count)
            
            
            global_value['global_count'] = sum_of_local_count     
            
            scaling_factor=local_count/global_value['global_count']
            scaled_weights = scale_model_weights(local_weight, scaling_factor)
            scaled_local_weight_list.append(scaled_weights)
            
            global_value['scale_statue'] = True 
            global_value['data_statue'] = True
            break
        elif (global_value['data_statue'] == True and global_value['scale_statue'] == True):
            scaling_factor=local_count/global_value['global_count']
            scaled_weights =scale_model_weights(local_weight, scaling_factor)
            scaled_local_weight_list.append(scaled_weights)

            break
        time.sleep(1)
```
After finishing calculate the weight, the server has to clear the data to ensure the next FL round is coooect.
Then return the weight to clients.
```
        clients_local_count.clear()
        scaled_local_weight_list.clear()
        
        return jsonify({'result': (global_value['average_weights'])})
```
After the all FL rounds finish, the clients will post a signal to server.
When the number of signal equal to NUM_OF_CLIENTS, the server will shutdown.
```
@app.route('/shutdown', methods=['GET'])
def shutdown_server():
    global_value['shutdown'] +=1 
    with shutdown_lock:
        while True:
            if(global_value['shutdown'] == NUM_OF_CLIENTS):
                os._exit(0)
                return 'Server shutting down...'
            time.sleep(1)
```


Client
---
This part initializes the dataset you use. In this example we use the hemodialysis information.
```
normal_url='<change yourself>' 
abnormal_url='<change yourself>'

normal_data = pd.read_csv(normal_url)		#init data, you should change method by your own data
abnormal_data = pd.read_csv(abnormal_url)
num_features = len(normal_data.columns)

normal_label = np.array([[1, 0]] * len(normal_data))
abnormal_label = np.array([[0, 1]] * len(abnormal_data))


data = np.vstack((normal_data, abnormal_data))
data_label = np.vstack((normal_label, abnormal_label))


shuffler = np.random.permutation(len(data))
data = data[shuffler]
data_label = data_label[shuffler]


data = data.reshape(len(data), num_features, 1)
data_label = data_label.reshape(len(data_label), 2)


full_data = list(zip(data, data_label))
data_length=len(full_data)
```
The model clients use. You can change your method here.
```
class SimpleMLP:
    @staticmethod
    def build(shape, classes):
        model = Sequential()
        model.add(Conv1D(filters=4, kernel_size=3, input_shape=(17,1)))
        model.add(MaxPooling1D(3))
        model.add(Flatten())
        model.add(Dense(8, activation="relu"))
        model.add(Dense(2, activation = 'softmax'))

        return model
```
This part is not a correct FL code because client should has it's own data. It's a way to get dataset from segment of full dataset.
```
if(batch==1):
    full_data=full_data[0:int(data_length/2)] #batch data
else:
    full_data=full_data[int(data_length/2):data_length] #The client should have its own data, not like this. It's a lazy method.
```

It's a url that client's connect to server in kubeflow(k8s). It's defining in k8s service.

```
server_url="http://http-service:5000/data"
```
The FL training begin and the number of round in this example is 5.

First round won't change model weight, but next rounds will use the "avg_weight" from server return.
After finish fit, client will pack it's information(like local weight) into json format for sending to server.
```
for comm_round in range(5):
    print('The ',comm_round+1, 'round')
    client_model = smlp_model.build(17, 1)
    client_model.compile(loss=loss, 
                  optimizer=optimizer, 
                  metrics=metrics)
    
    if(comm_round == 0):
        history = client_model.fit(dataset, epochs=50, verbose=1)
    else:
        client_model.set_weights(avg_weight)
        history = client_model.fit(dataset, epochs=50, verbose=1)
    
    local_weight = client_model.get_weights()
    local_weight = [np.array(w).tolist() for w in local_weight]
    
    client_data = {"local_count": local_count,'bs': bs, 'local_weight': json.dumps(local_weight)}
```

Then clients will use "request" to send data to server. But clients and server are established at the same time,
so use "while True" to check the server is already established.
After return, the client will get "avg_weight for next model fit.
```
while True:
    try:
        weight = (requests.post(server_url,data=client_data))
        
        if weight.status_code == 200:
            print(f"exist")

            break
        else:
            print(f"server error")

    except requests.exceptions.RequestException:

        print(f"not exist")
        
    time.sleep(5)
    
data = weight.json()
avg_weight = data.get('result')
avg_weight = json.loads(avg_weight)
avg_weight = [np.array(lst) for lst in avg_weight]
```

When all FL rounds are finish, clients will send shutdown request to server.
```
shutdown_url="http://http-service:5000/shutdown"    
try:
    response = requests.get(shutdown_url)
except requests.exceptions.ConnectionError:
    print('already shutdown')
```

Pipeline
---

Transfer the client and server funtion to kubeflow funtion by "func_to_container_op".
```
server_op=func_to_container_op(server,base_image='tensorflow/tensorflow',packages_to_install=['flask','pandas'])
client_op=func_to_container_op(client,base_image='tensorflow/tensorflow',packages_to_install=['requests','pandas'])
```
Create the k8s service to let clients can connect to server. To ensure the 'selector'(label) and 'targetPort' are match with server.

```
service = dsl.ResourceOp(
    name='http-service',
    k8s_resource={
        'apiVersion': 'v1',
        'kind': 'Service',
        'metadata': {
            'name': 'http-service'
        },
        'spec': {
            'selector': {
                'app': 'http-service'
            },
            'ports': [
                {
                    'protocol': 'TCP',
                    'port': 5000,
                    'targetPort': 8080
                }
            ]
        }
    }
)
```
Make the server task and add the labal. To ensure the "label" and the "container_port" are match with service.
```
server_task=server_op()
server_task.add_pod_label('app', 'http-service')
server_task.add_port(V1ContainerPort(name='my-port', container_port=8080))
server_task.set_cpu_request('0.2').set_cpu_limit('0.2')
server_task.after(service)
```
Delete the service after server shutdown. Use action="delete".
```
    delete_service = dsl.ResourceOp(
        name='delete-service',
        k8s_resource={
            'apiVersion': 'v1',
            'kind': 'Service',
            'metadata': {
                'name': 'http-service'
            },
            'spec': {
                'selector': {
                    'app': 'http-service'
                },
                'ports': [
                    {
                        'protocol': 'TCP',
                        'port': 80,
                        'targetPort': 8080
                    }
                ],
                'type': 'NodePort'  
            }
        },
        action="delete" #delete
    ).after(server_task)
```
Create clients. In this example is 2 clients.
```
client_task_1=client_op(1)
client_task_1.set_cpu_request('0.2').set_cpu_limit('0.2')
clienttask_2=client_op(2)
clienttask_2.set_cpu_request('0.2').set_cpu_limit('0.2')
```
