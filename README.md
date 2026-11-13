# Movie Recommender Chatbot

A command-line chatbot that recommends movies and keeps a watch list. It understands what you type using TF-IDF similarity against example phrases, keeps track of the conversation with a state machine, and recommends films from ratings scraped from Letterboxd. Built in Python as university coursework.

## What it can do

- Small talk, with a reply that depends on the sentiment of what you said
- Recognise your name from a sentence, and change it later
- Recommend five movies similar to one you like
- Add movies to a watch list, remove them and display it
- Answer questions about a movie: cast, director, runtime and genres
- List what it can do when you ask

## Example conversation

A real session, trimmed to the main flow (a few turns are left out). Lines starting with `>` are what I typed.

```text
Hello what is your name
> my name is alex
Hello Alex what can I do for you?
> recommend me a movie
Give me the name of a movie you like
> lego
Here are the results found:

	1. the lego story
	2. the lego movie
	3. the lego batman movie
	4. lego jack stone

Could you please specify the movie or the number on the list
> the lego movie
Is this the movie you were looking for

	the lego movie

> yes
Generating recommendations...
Alex, these are my top five movie recommendations:

	1. the lego batman movie
	2. wreck it ralph
	3. ant man
	4. monsters university
	5. avengers age of ultron

Would you like to add any of these to your watch list
> who is the director
Which of the recommended movies would you like to know about?
> 2
The director(s) of wreck it ralph are:
	*Rich Moore
Would you like to add a movie to your watch list?
> yes
What movie would you like to add to your watch list?
> 2
This movie has been added to your watch list
```

## How it works

**Dialogue state.** `main.py` is a state machine. In the base state it works out what you want, then hands over to a "transaction" (recommend, add to watch list, remove, display watch list, change name). Each transaction has its own sub-states, such as asking for a movie name, confirming a search result and asking a follow-up question. You can leave any of them by saying something like "cancel".

![Main loop flow chart](docs/main_loop_flowchart.png)

**Intent matching.** Each intent has a file of example phrases in `corpus/` (small talk, Q&A, discoverability, names, watch list commands and so on). Your input is turned into a TF-IDF vector and compared with each file using cosine similarity. The intent with the highest score wins, and thresholds I tuned by hand decide when to give up and say it didn't understand.

**Movie search.** Movie titles get the same treatment. Your text is matched against the titles, and results above a similarity threshold are listed. If several match, you can pick one by name or by its number in the list.

**Recommendations.** This is item-based collaborative filtering. The scraped ratings form a user-item matrix, each movie is a column of ratings, and the movies closest to your chosen one (by Euclidean distance) are recommended, skipping anything already on your watch list.

**Sentiment and names.** Sentiment comes from a lexicon of positive and negative words, so the generic small-talk reply can be positive, negative or neutral. Names are pulled out of a sentence with NLTK's named entity recogniser, then checked against a list of known names.

**Movie details.** Cast, director, runtime and genres are fetched live from Letterboxd when you ask.

## Data

The ratings data isn't included in the repo.

- **Ratings:** `get_data()` in `Initialise.py` scrapes the rated films of up to about 1,500 of Letterboxd's most popular members (skipping anyone with a very large film list), then saves them as `movie_data.joblib` and writes the movie list to `corpus/movies.txt`. It takes several hours, so it's commented out and only needs to run once. <!-- TODO: add how many users and movies you ended up with -->
- **Phrase files (included):** the files in `corpus/` hold the example phrases for each intent. I wrote most of them, and used ChatGPT to generate some extra rows so that a wider variety of inputs would match.
- **Sentiment lexicon (not included):** `corpus/subj_analysis.txt` is built from the [MPQA Subjectivity Lexicon](https://mpqa.cs.pitt.edu/lexicons/subj_lexicon/) (Wilson, Wiebe and Hoffmann, 2005). Download it yourself and save it as a CSV with a `Word` and a `Priorpolarity` column, which is what `main.py` reads.

## Running it

```bash
pip install pandas requests beautifulsoup4 joblib nltk scikit-learn
mkdir -p corpus count_vectorizers
```

Download the NLTK resources it needs:

```python
import nltk
for r in ["punkt", "punkt_tab", "averaged_perceptron_tagger", "averaged_perceptron_tagger_eng",
          "maxent_ne_chunker", "maxent_ne_chunker_tab", "words", "wordnet", "stopwords"]:
    nltk.download(r)
```

The phrase files are already in `corpus/`. Add `corpus/subj_analysis.txt` (see above), then generate the ratings data once (this also writes `corpus/movies.txt`):

```bash
python -c "from Initialise import get_data; get_data()"
python main.py
```

Each start-up rebuilds the TF-IDF vectorisers from `corpus/` and loads the ratings matrix.

## Evaluation

I measured usability with the System Usability Scale (SUS). One tester used the chatbot and then answered the ten questions, giving a score of **70 out of 100** (the usual average is 68). With a single tester this is only an indication.

What they ran into:
- Some inputs weren't recognised, and the chatbot asked them to try again.
- If a question already contained the movie, for example "What is the genre of movie 2", it recognised the question but still asked which movie they meant.
- Generating recommendations took around 15 seconds, much longer than any other response.

Ideas I haven't tried: augmenting the phrase files with variations and synonyms, extracting the movie from the question itself, and listing what the chatbot can do when it starts.

## Limitations

- It's a rule-based system with hand-tuned thresholds, so it only understands phrasings close to the examples in `corpus/`.
- Unrated movies count as a rating of zero, and similarity is plain Euclidean distance, so recommendations favour popular films. I haven't evaluated them against any ground truth.
- Movie details come from parsing Letterboxd's HTML with string searches, which breaks whenever the site changes.
- The watch list and conversation state live in memory, so they're lost when you quit.
- Recommendations are slow because the distance to every other movie is computed one at a time in a Python loop. A vectorised version should be much faster.
- Letterboxd's terms apply to anyone running the scraper, so check them first.

## Files

| File | Purpose |
|---|---|
| `main.py` | Conversation loop, state machine and all the transactions |
| `Initialise.py` | Scraper, TF-IDF vectoriser creation and the user-item matrix |
