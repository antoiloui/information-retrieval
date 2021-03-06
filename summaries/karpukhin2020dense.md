## Dense Passage Retrieval for Open-Domain Question Answering

+++++++++++++++++++++++++++++++  
<ins>Authors</ins>: V. Karpukhin et al.  
<ins>Date</ins>: 2020-04  
<ins>Tags</ins>:   
+++++++++++++++++++++++++++++++  


## Intro

- Open-domain question answering (QA) is a task that answers factoid questions using a large collection of documents.
- The advances of reading comprehension models suggest a simple two-stage framework:
  1. a **context retriever** first selects a small subset of passages where some of them contain the answer to the question;
  2. a **machine reader** can thoroughly examine the retrieved contexts and identify the correct answer.
- Retrieval in open-domain QA is usually implemented using **TF-IDF** or **BM25**, which matches keywords efficiently with an inverted index and can be seen as representing the question and context in high-dimensional sparse vectors (with weighting). Conversely, the dense latent semantic encoding is complementary to sparse representations by design (e.g., synonyms or paraphrases that consist of completely different tokens may still be mapped to vectors close to each other). Retrieval with dense encodings can be doneefficiently using **maximum inner product search** (MIPS) algorithms.
- However, it is generally believed that learning a good dense vector representation needs a large number of labeled pairs of question and contexts. Dense retrieval methods have thus never be shown to outperform TF-IDF/BM25 for opendomain QA before [ORQA](./lee2019latent.md).


## Contributions

- Although ORQA successfully demonstrates that dense retrieval can outperform BM25, it also suffers from two weaknesses:
  1. First, inverse cloze task (ICT) pretraining is computationally intensive and it is not completely clear that regular sentences are good surrogates of questions in the objective function.
  2. Second, because the context encoder is not fine-tuned using pairs of questions and answers, the corresponding representations could be suboptimal.
- In this paper, they address the question: can we train a better dense embedding model using only pairs of questions and passages (or answers), without additional pretraining? In particular, they focus their research on improving the retrieval component in open-domain QA.
- By leveraging the BERT pretrained model and a dual-encoder architecture, they focus on developing the right training scheme using a relatively small number of question and passage pairs. The embedding is optimized for maximizing inner products of the question and relevant passage vectors, with an objective comparing all pairs of questions and passages in a batch.
- Their *Dense Passage Retriever* (DPR) outperforms BM25 by a large margin but also results in a substantial improvement on the end-to-end QA accuracy compared to *ORQA*.

***

## 1. Dense Passage Retriever (DPR)

- <ins>Overview</ins>
  - DPR uses two different **dense encoders** (BERT-base networks uncased):
    1. The passage encoder *E<sub>P</sub>(.)*, which maps any text **passage** to a *d*-dimensional real-valued vectors. At run-time, the passage encoder is used once to encode all the passages offline, whose representations are then indexed using FAISS.
    2. The question encoder *E<sub>Q</sub>(.)*, which maps the input **question** to a *d*-dimensional vector. At run-time, the *k* passages of which vectors are the closest to the question vector are retrieved. They define the similarity between the question and the passage using the **dot product** of their vectors (NB: they also tested cosine and L2 distances and all performed similarly, dot product leading to slightly better performance).
  - For both encoders, the output corresponds to the representation of the [CLS] token.

- <ins>Training</ins>
  - Training the encoders so that the dot-product similarity becomes a good ranking function for retrieval is essentially a **metric learning problem**. The goal is to create a vector space such that relevant pairs of questions and passages will have smaller distance than the irrelevant ones, by learning a better embedding function.
  - Each training instance contains one question and one relevant (positive) passage along with *n* irrelevant (negative) passages. They then optimize the loss function as the negative log likelihood of the positive passage.
  - Here, they consider three types of negatives:
    1. Random: any random passage from the corpus; 
    2. BM25: top passages returned by BM25 which don’t contain the answer but match most question tokens;
    3. Gold: positive passages paired with other questions which appear in the training set.
  - Assume that a mini-batch has *B* questions and each one is associated with a relevant passage. Let **Q** and **P** be the *(B x d)* matrix of question and passage embeddings in a batch of size *B*. Then, **S** = **QP** is a *(B x B)* QPT is a matrix of similarity scores, where each row of which corresponds to a question, paired with *B* passages. In this way, they reuse computation and effectively train on *B<sup>2</sup> (qi, pj)* question/passage pairs in each batch.
  


## 2. Experiments on Passage Retrieval

- <ins>Data</ins>
  - QA: Natural Questions (NQ), TriviaQA, WebQuestions (WQ), CuratedTREC (TREC), SQuAD v1.1.
  - Retrieval corpus: used the English Wikipedia dump from Dec. 20, 2018 where each article is split into blocks of 100 words, resulting in ~21M passages. To select the positive passages, because only pairs of questions and answers are provided in TriviaQA, TREC and WebQuestions, they used the highest-ranked passage from BM25 that contains the answer as the positive passage.
- <ins>Results</ins>
  - They compared DPR to BM25 and BM25+DPR using a linear combination of their scores as the new ranking function.
  - With the exception of SQuAD, DPR performs consistently better than BM25 on all datasets (the gap is especially large when *k* is small). Results can be improved further in some cases by combining DPR with BM25.
  - Probably 2 reasons why performance on SQUAD is lower: (1) the annotators wrote questions after seeing the passage (there is then a high lexical overlap between passages and questions, which gives BM25 a clear advantage); (2) the data was collected from only 500+ Wikipedia articles and thus the distribution of training examples is extremely biased.
- <ins>Ablation studies</ins>
  - **Sample efficiency: how many training examples are needed to achieve good passage retrieval performance?**
    - Result: a dense passage retriever trained using only 1,000 examples already outperforms BM25.
    - This suggests that with a general pretrained language model, it is possible to train a high-quality dense retriever with a small number of question–passage pairs.
    - Adding more training examples (from 1k to 59k) further improves the retrieval accuracy consistently.
  - **In-batch negative training**: testing different training shcemes.
    - They found that the choice of negatives — random, BM25 or gold passages (positive passages from other questions) — does not impact the top-k accuracy much in this setting when k > 20.
    - They also explored in-batch negative training with additional “hard” negative passages that have high BM25 scores given the question, but do not contain the answer string. They found that adding a single BM25 negative passage improves the result substantially while adding two does not help further.
  - **Similarity and loss**:
    - Besides **dot product**, **cosine** and **Euclidean L2** distance are also commonly used as decomposable similarity functions. They found that L2 performs comparable to dot product, and both of them are superior to cosine.
    - In addition to **negative log-likelihood**, a popular option for ranking is **triplet loss**, which compares a positive passage and a negative one directly with respect to a question. Their experiments show that using triplet loss does not affect the results much.

    
