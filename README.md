# Fitness Assistant

<p align="center">
  <img src="images/banner.jpg">
</p>

## Problem description

Staying consistent with fitness routines is challenging,
especially for beginners. Gyms can be intimidating, and
personal trainers aren't always available.

The Fitness Assistant provides a conversational AI that helps
users choose exercises and find alternatives, making fitness
more manageable.


## Project overview

The Fitness Assistant is a RAG application for assisting users
with their fitness routines.

The main use cases include:

1. Exercise Selection: recommend exercises based type of activity, targeted muscle groups, or available equipment
2. Exercise Replacement: replace an exercise with a suitable alternatives
3. Exercise Instructions: perform a specific exercise
4. Conversational Interaction: make it easy to get the information without sifting through manuals or websites

## Dataset

The dataset used in this project contains information about various exercises, including:

- **Exercise Name:** The name of the exercise (e.g., Push-Ups, Squats).
- **Type of Activity:** The general category of the exercise (e.g., Strength, Mobility, Cardio).
- **Type of Equipment:** The equipment needed for the exercise (e.g., Bodyweight, Dumbbells, Kettlebell).
- **Body Part:** The part of the body primarily targeted by the exercise (e.g., Upper Body, Core, Lower Body).
- **Type:** The movement type (e.g., Push, Pull, Hold, Stretch).
- **Muscle Groups Activated:** The specific muscles that are engaged during the exercise (e.g., Pectorals, Triceps, Quadriceps).
- **Instructions:** Step-by-step guidance on how to perform the exercise correctly.

The dataset was generated using ChatGPT and contains 207 records. It serves as the foundation for the Fitness Assistant's exercise recommendations and instructional support.

You can find the data in [`data/data.csv`](data/data.csv).


## Technologies

* Python 3.12
* Docker and docker-compose for containerization
* [Minsearch](https://github.com/alexeygrigorev/minsearch) - for full-text search
* Flask as the API interface (see [Background](#background) for more information on Flask)
* Grafana for monitoring and Postgres as backend for it
* OpenAI as an LLM


## Preparation

Because we use OpenAI, you need to provide the API key:

* Install `direnv`. If you use Ubuntu, it's `sudo apt install direnv`
* Copy `.envrc_template` into `.envrc` and insert your key there
* It's recommended to create a new project and use a separate key
* Run `direnv allow` to load the key into your environment

For dependency management, we use pipenv, so you need to install
it:

```bash
pip install pipenv
```

Once you have it, install the app dependencies:

```bash
pipenv install --dev
```

## Running it 

The easiest way to run this application is with docker-compose:

```bash
docker-compose up
```


### Database configuration

When the application starts for the first time, we need 
to initialize the database.

Run the [`db_prep.py`](fitness_assistant/db_prep.py) script:

```bash
cd fitness_assistant

pipenv shell

export POSTGRES_HOST=localhost
python db_prep.py
```

Note: you do this after you run `docker-compose up`.

You can check the content of the database by running `pgcli` (already installed with pipenv):

```bash
pipenv run pgcli -h localhost -U your_username -d course_assistant -W
```

You can see the schema using the `\d` command:

```sql
\d conversations;
```

And select from this table:

```sql
select * from conversations;
```

### Time and timezone configuration

When inserting the logs into the database, we need to
make sure the timestamps we insert are correct - otherwise
they won't be displayed in Grafana.

When you start the application, you will see the following
in your logs:

```
Database timezone: Etc/UTC
Database current time (UTC): 2024-08-24 06:43:12.169624+00:00
Database current time (Europe/Berlin): 2024-08-24 08:43:12.169624+02:00
Python current time: 2024-08-24 08:43:12.170246+02:00
Inserted time (UTC): 2024-08-24 06:43:12.170246+00:00
Inserted time (Europe/Berlin): 2024-08-24 08:43:12.170246+02:00
Selected time (UTC): 2024-08-24 06:43:12.170246+00:00
Selected time (Europe/Berlin): 2024-08-24 08:43:12.170246+02:00
```

Make sure the time is correct.

You can change the timezome bt replacing `TZ` in `.env`.

On some systems, specifically WSL, the clock in Docker
may get out of sync with the host system. You can check that
by running

```bash
docker run ubuntu date
```

If the time doesn't match yours, you need to sync the clock:

```bash
wsl 

sudo apt install ntpdate
sudo ntpdate time.windows.com
```

Note that the time is in UTC.

After that, start the application (and the database) again.


### Running locally

For running the application locally, start docker-compose, but
stop the app:

```bash
docker-compose up postgres grafana
```

If you previously started all the applications with `docker-compose up`, you need to stop the `app`:

```bash
docker-compose stop app
```

Now run the app in your host machine:

```bash
pipenv shell
cd fitness_assistant

export POSTGRES_HOST=localhost
python app.py
```

### Running it with Docker (without compose)

Sometimes you want to run the application in docker
without docker-compose, e.g. for debugging purposes.

First, prepare the env by running docker-compose like 
in the previous session.

Next, build the image:

```bash
docker build -t fitness-assistant .
```

And run it:

```bash
docker run -it --rm \
    --network="fitness-assistant_default" \
    --env-file=".env" \
    -e OPENAI_API_KEY=${OPENAI_API_KEY} \
    -e DATA_PATH="data/data.csv"\
    -p 5000:5000 \
    fitness-assistant
```

## Using the application

When the application is running, we can use requests for
sending questions - use [test.py](test.py) for testing it:

```bash
pipenv run python test.py
```

We can also use `curl` for interacting with the API:

```bash
URL=http://localhost:5000

QUESTION="Is the Lat Pulldown considered a strength training activity, and if so, why?"

DATA='{
    "question": "'${QUESTION}'"
}'

curl -X POST \
  -H "Content-Type: application/json" \
  -d "${DATA}" \
  ${URL}/question
```

You will see sonething like the following in the response:

```json
{
  "answer": "Yes, the Lat Pulldown is considered a strength training activity. This classification is due to it targeting specific muscle groups, specifically the Latissimus Dorsi and Biceps, which are essential for building upper body strength. The exercise utilizes a machine, allowing for controlled resistance during the pulling action, which is a hallmark of strength training.",
  "conversation_id": "4e1cef04-bfd9-4a2c-9cdd-2771d8f70e4d",
  "question": "Is the Lat Pulldown considered a strength training activity, and if so, why?"
}
```

Sending feedback:

```bash
ID="4e1cef04-bfd9-4a2c-9cdd-2771d8f70e4d"

URL=http://localhost:5000

FEEDBACK_DATA='{
    "conversation_id": "'${ID}'",
    "feedback": 1
}'

curl -X POST \
  -H "Content-Type: application/json" \
  -d "${FEEDBACK_DATA}" \
  ${URL}/feedback
```

After sending it, you'll receive the acknowledgement:


```json
{
  "message": "Feedback received for conversation 4e1cef04-bfd9-4a2c-9cdd-2771d8f70e4d: 1"
}
```


## Code

The code for the application is in the
[`fitness_assistant`](fitness_assistant/) folder:

- [`app.py`](fitness_assistant/app.py)
- [`ingest.py`](fitness_assistant/ingest.py)
- [`minsearch.py`](fitness_assistant/minsearch.py)
- [`rag.py`](fitness_assistant/rag.py)
- `db_prep.py`
- `db.py`


### Interface

We use Flask for serving the application as API.

Refer to ["Using the application" section](#using-the-application) for examples on how to interact with the application.


### Ingestion

The ingestion script is in [`ingest.py`](fitness_assistant/ingest.py).

Because we use an in-memory database minsearch as our
knowledge base, we run the ingestion script on the startup
of the application.

It's run inside [`rag.py`](fitness_assistant/rag.py) when we import it


## Experiments

For experiments, we use Jupyter notebooks. They are in the [`notebooks`](notebooks/) folder

To start jupyter, run:

```bash
cd notebooks
pipenv run jupyter notebook
```

We have the following notebooks:

* [`rag-test.ipynb`](notebooks/rag-test.ipynb): the RAG flow and evaluating the system
* [`evaluation-data-generation.ipynb`](notebooks/evaluation-data-generation.ipynb): generating the ground truth dataset for retrieval evaluation


### Retrieval evaluation

The basic approach - using `minsearch` without any boosting - gave the following metrics:

* hit_rate: 94%
* MRR: 82%

The improved vesion (with tuned boosting):

* hit_rate: 94%
* MRR: 90%

The best boosting parameters:

```python
boost = {
    'exercise_name': 2.11,
    'type_of_activity': 1.46,
    'type_of_equipment': 0.65,
    'body_part': 2.65,
    'type': 1.31,
    'muscle_groups_activated': 2.54,
    'instructions': 0.74
}
```

### RAG flow evaluation

We used the LLM-as-a-Judge metric to evaluate the quality
of our RAG flow

For `gpt-4o-mini`, in a sample with 200 records, we had:

* 167 (83%) `RELEVANT`
* 30 (15%) `PARTLY_RELEVANT`
* 3 (1.5%) `NON_RELEVANT`

We also tested `gpt-4o`:

* 168 (84%) `RELEVANT`
* 30 (15%) `PARTLY_RELEVANT`
* 2 (1%) `NON_RELEVANT`

The difference is far from significant, so we went with `gpt-4o-mini`.


## Monitoring

We use Grafana for monitoring the application. 

It's accessible at [localhost:3000](http://localhost:3000):

* login: "admin"
* password: "admin"

### Dashboards

<p align="center">
  <img src="images/dash.png">
</p>

The monitoring dashboard contains several panels:

1. **Last 5 Conversations (Table)**: Displays a table showing the five most recent conversations, including details such as the question, answer, relevance, and timestamp. This panel helps monitor recent interactions with users.

2. **+1/-1 (Pie Chart)**: A pie chart that visualizes the feedback from users, showing the count of positive (thumbs up) and negative (thumbs down) feedback received. This panel helps track user satisfaction.

3. **Relevancy (Gauge)**: A gauge chart representing the relevance of the responses provided during conversations. The chart categorizes relevance and indicates thresholds using different colors to highlight varying levels of response quality.

4. **OpenAI Cost (Time Series)**: A time series line chart depicting the cost associated with OpenAI usage over time. This panel helps monitor and analyze the expenditure linked to the AI model's usage.

5. **Tokens (Time Series)**: Another time series chart that tracks the number of tokens used in conversations over time. This helps to understand the usage patterns and the volume of data processed.

6. **Model Used (Bar Chart)**: A bar chart displaying the count of conversations based on the different models used. This panel provides insights into which AI models are most frequently used.

7. **Response Time (Time Series)**: A time series chart showing the response time of conversations over time. This panel is useful for identifying performance issues and ensuring the system's responsiveness.

### Setting up Grafana

All the Grafana configuration is in the
[`grafana`](grafana/) folder:

* [`init.py`](grafana/init.py) - for initializing the datasource and the dashboard
* [`dashboard.json`](grafana/dashboard.json) - the actual dashboard (taken from LLM Zoomcamp without changes)

To initialize the dashboard, first make sure grafana is running
(it starts automatically when you do `docker-compose up`)

And then run:

```bash
cd grafana
python init.py
```

Then go to [localhost:3000](http://localhost:3000):

* login: "admin"
* password: "admin"

When prompted, keep "admin" as the new password.


## Background

Here we provide background on some tech not used in the course
and links for futher reading.

### Flask

We use Flask for creating the API interface for our application.
It's a web application framework for Python: we can easily create
and endpoint for asking questions and use web clients (like
`curl` or `requests`) for communicating with it.

In our case, we can send the question to `http://localhost:5000/question`.

For more information, visit the [official Flask documentation](https://flask.palletsprojects.com/).