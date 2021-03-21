# Mahout Recommender - Big Data Assignment
Building a Recommender with Apache Mahout on Amazon Elastic MapReduce (EMR)

1. Sign up for an AWS account and start up an EMR cluster.

2. Get the MovieLens data.
```
wget http://files.grouplens.org/datasets/movielens/ml-1m.zip
```

3. Unzip the file.

```
unzip ml-1m.zip
```

4. Convert ratings.dat, trade “::” for “,”, and take only the first three columns:

```
cat ml-1m/ratings.dat | sed 's/::/,/g' | cut -f1-3 -d, > ratings.csv
```

5. Put ratings file into HDFS:

```
hadoop fs -put ratings.csv /ratings.csv
```

6. Run the recommender job:

```
mahout recommenditembased --input /ratings.csv --output recommendations --numRecommendations 10 --outputPathForSimilarityMatrix similarity-matrix --similarityClassname SIMILARITY_COSINE
```

7. Look for the results in the part-files containing the recommendations:

```
hadoop fs -ls recommendations
hadoop fs -cat recommendations/part-r-00000 | head
```

The first number is a user id, and the key-value pairs inside the brackets are movie-id:recommendation-strength tuples.

8. Next, we’ll use this lookup file in a simple web service that returns movie recommendations for any given user.

Get Twisted, and Klein and Redis modules for Python.

```
sudo pip3 install twisted
sudo pip3 install klein
sudo pip3 install redis
```

9. Install Redis and start up the server.

```
wget http://download.redis.io/releases/redis-2.8.7.tar.gz
tar xzf redis-2.8.7.tar.gz
cd redis-2.8.7
make
./src/redis-server &
```
        
10. Build a web service that pulls the recommendations into Redis and responds to queries.
Put the following into a file, e.g., “hello.py”

```
from klein import run, route
import redis
import os

# Start up a Redis instance
r = redis.StrictRedis(host='localhost', port=6379, db=0)

# Pull out all the recommendations from HDFS
p = os.popen("hadoop fs -cat recommendations/part*")

# Load the recommendations into Redis
for i in p:

  # Split recommendations into key of user id
  # and value of recommendations
  # E.g., 35^I[2067:5.0,17:5.0,1041:5.0,2068:5.0,2087:5.0,
  #       1036:5.0,900:5.0,1:5.0,081:5.0,3135:5.0]$
  k,v = i.split('\t')

  # Put key, value into Redis
  r.set(k,v)

# Establish an endpoint that takes in user id in the path
@route('/<string:id>')

def recs(request, id):
  # Get recommendations for this user
  v = r.get(id)
  return 'The recommendations for user '+id+' are '+v.decode()


# Make a default endpoint
@route('/')

def home(request):
  return 'Please add a user id to the URL, e.g. http://localhost:8083/1234n'

# Start up a listener on port 8083
run("localhost", 8083)
```

11. Start the web service.
```
twistd -noy hello.py &
```

12. Test the web service with user id “40”:
```
curl localhost:8083/40
```

    You should see a response like this:

    The recommendations for user 40 are [2376:5.0,2377:5.0,594:5.0,3034:5.0,3168:5.0,3498:5.0,3628:5.0,1253:5.0,3035:5.0,3494:5.0]

