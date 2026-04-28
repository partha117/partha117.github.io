---
permalink: /
title: "Partha Chakraborty"
description: "Applied Scientist building retrieval and ranking systems at scale. Work on hybrid search, bi-encoder training, and production ML evaluation at Coalition Inc."
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---
I build retrieval and ranking systems that work at scale. My work sits at the intersection of classical information retrieval and modern neural approaches: fine-tuning embedding models, designing hybrid search pipelines, and building the evaluation infrastructure that tells you whether search is actually getting better.

At Coalition Inc., I work on enterprise search and knowledge retrieval: a production hybrid search system combining dense vector and text indexes across tens of thousands of sources, a fine-tuned bi-encoder for semantic entity resolution, and an evaluation platform covering both classical search metrics and LLM-based quality signals. Before that, I fine-tuned vision-language models at Huawei Canada and built a learning-to-rank system for ad retrieval across 21 Amazon marketplaces.

My research at the University of Waterloo explored the same problem space from the academic side. RLocator applied reinforcement learning to learning-to-rank. BLAZE used hybrid retrieval with Reciprocal Rank Fusion for zero-shot code search. The embedding design work investigated what training strategies actually produce better bi-encoders. Adversarial techniques turned out to matter more than architecture choices.

Research Interests
======

**Search & Ranking Systems**
Learning-to-rank, hybrid retrieval, dense and sparse signal fusion, and relevance evaluation. Interested in how classical IR techniques and neural approaches complement each other, and in evaluation frameworks that measure retrieval quality rigorously rather than by proxy metrics.

**Representation Learning for Retrieval**
Embedding model training across bi-encoder and cross-encoder architectures, hard negative mining strategies, and domain adaptation. Particularly interested in cases where off-the-shelf embeddings fail and what it takes to fix them.

**Production AI Systems**
LLM-based agents, RAG pipelines, multimodal models, and the evaluation and safety infrastructure that makes them trustworthy at scale. Search is often the retrieval backbone of these systems, but the system-level challenges go beyond retrieval quality alone.


<div id="work_experience">
  <h1>Work Experience</h1>
  <div>
    <div style="display: flex; justify-content: space-between; align-items: center;">
        <p style="font-size: 20px; font-weight: bold; margin: 0;">Applied Scientist</p>
        <p style="font-size: 18px; font-style: italic; margin: 0;">April 2025 – Present</p>
    </div>
    <div class="archive__proj__row">
      <div class="archive__proj__right">
        <p style="text-align: justify;">
        <b>Enterprise Search and Retrieval at Scale</b>
        <br>Built the search and knowledge retrieval infrastructure powering Coalition's enterprise AI platform. Core work includes a production hybrid search system combining Milvus dense vector indexes and text indexes across ~16K sources, a fine-tuned sentence transformer bi-encoder for semantic entity resolution trained with online hard negative mining, and a dual-layer evaluation platform tracking classical search metrics (MAP, MRR, nDCG, Hit@K) alongside LLM-based quality signals including faithfulness, completeness, and context relevance.
        <br><br>Also built an unsupervised intent discovery pipeline using embeddings, UMAP, and HDBSCAN to surface emerging user issues from unstructured conversations — moving beyond rigid predefined categories to identify real product gaps. The platform monitors 50+ agents and 30+ models with continuous evaluation runs every three hours, enabling regression detection and A/B testing across LLM providers.
        </p>
      </div>
      <div class="archive__proj__left">
        <div>
            <img src="images/coalition.png"> 
        </div>
      </div>
    </div>
  </div>

  <div>
    <div style="display: flex; justify-content: space-between; align-items: center;">
        <p style="font-size: 20px; font-weight: bold; margin: 0;">Deep Learning Engineer</p>
        <p style="font-size: 18px; font-style: italic; margin: 0;">November 2024 – April 2025</p>
    </div>
    <div class="archive__proj__row">
      <div class="archive__proj__right">
        <p style="text-align: justify;">
        <b>Vision-Language Models for AI-Generated Content</b>
        <br>Fine-tuned a vision-language model to generate photography composition instructions, achieving a 3% improvement in aesthetic classification accuracy over state-of-the-art. Improved model alignment through mixed preference optimization, increasing user acceptance by 10% in end-user evaluation. Reduced inference pipeline latency by 35% through dynamic quantization, enabling efficient large-scale deployment of the multimodal system.
        </p>
      </div>
      <div class="archive__proj__left">
        <div>
            <img src="images/huawei.png"> 
        </div>
      </div>
    </div>
  </div>

  <div>
    <div style="display: flex; justify-content: space-between; align-items: center;">
        <p style="font-size: 20px; font-weight: bold; margin: 0;">Applied Scientist II Intern</p>
        <p style="font-size: 18px; font-style: italic; margin: 0;">September 2023 – December 2023</p>
    </div>
    <div class="archive__proj__row">
      <div class="archive__proj__right">
        <p style="text-align: justify;">
        <b>Learning-to-Rank for Ad Retrieval</b>
        <br>Built a large-scale learning-to-rank model for ad retrieval across 21 Amazon marketplaces, improving ad-customer alignment by <b>3%</b> through five novel engineered features and multi-objective ranking optimization. Designed ranking strategies that balanced competing signals across a high-traffic production advertising system.
        </p>
      </div>
      <div class="archive__proj__left">
        <div>
            <img src="images/amazon_ads.png"> 
        </div>
      </div>
    </div>
  </div>

  <div>
    <div style="display: flex; justify-content: space-between; align-items: center;">
        <p style="font-size: 20px; font-weight: bold; margin: 0;">Software Engineer</p>
        <p style="font-size: 18px; font-style: italic; margin: 0;">November 2018 – September 2020</p>
    </div>
    <div class="archive__proj__row">
      <div class="archive__proj__right">
        <p style="text-align: justify;">
        <b>Search, Personalization, and Platform Infrastructure</b>
        <br>Built a personalized search system for address and point-of-interest retrieval, incorporating user attributes, history, and preferences into ranking. Achieved a <b>21.4% increase in CTR</b> and a <b>5% reduction in search abandonment</b>. Integrated intent-based ad placement into the search experience, generating <b>$35K in revenue within two months</b> of launch.
        <br><br>Also built a microservices-based streaming platform serving ~15K IoT devices using Redis and Elasticsearch, and a distributed data pipeline for validation and sanitization that reduced manual processing effort by 30%.
        </p>
      </div>
      <div class="archive__proj__left">
        <div>
            <img src="images/dingi.png"> 
        </div>
      </div>
    </div>
  </div>
</div>

<div id="projects">
  <h1>Projects</h1>

  <div class="archive__proj__row">
    <div class="archive__proj__right">
      <p style="text-align: justify;">
      <b><a href="https://doi.org/10.1109/TSE.2025.3579574" target="_blank">BLAZE: Hybrid Retrieval for Bug Localization</a></b>
      <br>A zero-shot hybrid retrieval system combining dense and sparse signals via Reciprocal Rank Fusion. The embedding model was trained with in-batch hard negative mining. Outperforms OpenAI's third-generation embedding model by up to 38% on bug localization benchmarks — zero-shot, no task-specific fine-tuning.
      <br><em>IEEE Transactions on Software Engineering, 2025</em>
      </p>
    </div>
    <div class="archive__proj__left">
      <div>
          <img src="images/blaze.png"> 
      </div>
    </div>
  </div>

  <div class="archive__proj__row">
    <div class="archive__proj__right">
      <p style="text-align: justify;">
      <b><a href="https://ieeexplore.ieee.org/document/10659717" target="_blank">RLocator: Reinforcement Learning for Learning-to-Rank</a></b>
      <br>A learning-to-rank model using actor-critic reinforcement learning to rank source code files given a bug report. Trained on developer feedback signals rather than static labels. Outperforms BM25 and neural baselines by 13% MAP.
      <br><em>IEEE Transactions on Software Engineering, 2024</em>
      </p>
    </div>
    <div class="archive__proj__left">
      <div>
          <img src="images/rlocator.png"> 
      </div>
    </div>
  </div>

  <div class="archive__proj__row">
    <div class="archive__proj__right">
      <p style="text-align: justify;">
      <b><a href="https://dl.acm.org/doi/abs/10.1145/3643787.3648028" target="_blank">Embedding Design Choices for Code-Language Retrieval</a></b>
      <br>Evaluated 32 embedding model configurations to identify what training strategies matter most for code-text retrieval. Key finding: adversarial training techniques produce more robust and generalizable models than architecture changes alone.
      <br><em>NLBSE Workshop, ACM/IEEE, 2024</em>
      </p>
    </div>
    <div class="archive__proj__left">
      <div>
          <img src="images/embedding.png"> 
      </div>
    </div>
  </div>

  <div class="archive__proj__row">
    <div class="archive__proj__right">
      <p style="text-align: justify;">
      <b><a href="https://ieeexplore.ieee.org/abstract/document/10587162" target="_blank">Vulnerability Detection on Realistic Datasets</a></b>
      <br>Showed that existing vulnerability detection models fail to generalize due to dataset curation bias. Proposed a new curation technique and benchmarked six models including CodeLlama and Mixtral. Models trained on the corrected dataset showed 30% improvement in generalization.
      <br><em>IEEE Transactions on Software Engineering, 2024</em>
      </p>
    </div>
    <div class="archive__proj__left">
      <div>
          <img src="images/realvul.png"> 
      </div>
    </div>
  </div>
</div>


<div id="publications">
  <h1>Publications</h1>
  <div>
    For latest publications, please visit <u><a href="https://scholar.google.ca/citations?user=YdI4XmUAAAAJ&hl=en" target="_blank">my Google Scholar profile</a>.</u>
    <br><br>
  </div>
  <div>
    <p class="publication">
      <b class="publication-title">BLAZE: Cross-Language and Cross-Project Bug Localization via Dynamic Chunking and Hard Example Learning</b>
      <b class="highlight-author">Partha Chakraborty</b>, 
      <span class="other-authors">Mahmoud Alfadel, and Meiyappan Nagappan</span>
      <br><span class="venue-year">IEEE Transactions on Software Engineering (<b>TSE</b>), 2025.</span>  
    </p>
    <p class="publication">
      <b class="publication-title">Rlocator: Reinforcement learning for bug localization</b>
      <b class="highlight-author">Partha Chakraborty</b>, 
      <span class="other-authors">Mahmoud Alfadel, and Meiyappan Nagappan</span>
      <br><span class="venue-year">IEEE Transactions on Software Engineering (<b>TSE</b>), 2024.</span>  
    </p>
    <p class="publication">
      <b class="publication-title">Revisiting the Performance of Deep Learning-Based Vulnerability Detection on Realistic Datasets</b>
      <b class="highlight-author">Partha Chakraborty</b>, 
      <span class="other-authors">Mahmoud Alfadel, Meiyappan Nagappan, and Shane McIntosh</span>
      <br><span class="venue-year">IEEE Transactions on Software Engineering (<b>TSE</b>), 2024.</span>  
    </p>
    <p class="publication">
      <b class="publication-title">Aligning Programming Language and Natural Language: Exploring Design Choices in Multi-Modal Transformer-Based Embedding for Bug Localization</b>
      <b class="highlight-author">Partha Chakraborty</b>, 
      <span class="other-authors">Venkatraman Arumugam, and Meiyappan Nagappan</span>
      <br><span class="venue-year">Third ACM/IEEE International Workshop on NL-based Software Engineering (<b>NLBSE</b>), 2024.</span>  
    </p>
    <p class="publication">
      <b class="publication-title">A Survey-Based Qualitative Study to Characterize Expectations of Software Developers from Five Stakeholders</b>
      <span class="other-authors">Khalid Hasan</span>
      <b class="highlight-author">Partha Chakraborty</b>, 
      <span class="other-authors">Rifat Shahriyar, Anindya Iqbal, and Gias Uddin</span>
      <br><span class="venue-year">Proceedings of the 15th ACM/IEEE International Symposium on Empirical Software Engineering and Measurement (<b>ESEM</b>), 2021.</span>  
    </p>
    <p class="publication">
      <b class="publication-title">How do developers discuss and support new programming languages in technical Q&A site? An empirical study of Go, Swift, and Rust in Stack Overflow</b>
      <b class="highlight-author">Partha Chakraborty</b>, 
      <span class="other-authors">Rifat Shahriyar, Anindya Iqbal, and Gias Uddin</span>
      <br><span class="venue-year">Information and Software Technology (<b>IST</b>), 2021.</span>  
    </p>
    <p class="publication">
      <b class="publication-title">Understanding the motivations, challenges and needs of blockchain software developers: A survey</b>
      <span class="other-authors">Amiangshu Bosu, Anindya Iqbal, Rifat Shahriyar, and </span>
      <b class="highlight-author">Partha Chakraborty</b> 
      <br><span class="venue-year">Empirical Software Engineering (<b>EMSE</b>), 2019.</span>  
    </p>
    <p class="publication">
      <b class="publication-title">Empirical Analysis of the Growth and Challenges of New Programming Languages</b>
      <b class="highlight-author">Partha Chakraborty</b>, 
      <span class="other-authors"> Rifat Shahriyar, and Anindya Iqbal</span>
      <br><span class="venue-year">IEEE 43rd Annual Computer Software and Applications Conference (<b>COMPSAC</b>), 2019.</span>  
    </p>
    <p class="publication">
      <b class="publication-title">Understanding the software development practices of blockchain projects: a survey</b>
      <b class="highlight-author">Partha Chakraborty</b>, 
      <span class="other-authors"> Rifat Shahriyar, Anindya Iqbal, and Amiangshu Bosu</span>
      <br><span class="venue-year">Proceedings of the 12th ACM/IEEE International Symposium on Empirical Software Engineering and Measurement (<b>ESEM</b>), 2018.</span>  
    </p>
  </div>
</div>

<div id="education">
  <h1>Education</h1>
  <h2>University of Waterloo</h2>
  <em>Ph.D.</em> in Computer Science &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; January 2021 - November 2024
  <p>
    <ul>
      <li>Specialization: Artificial Intelligence</li>
      <li>Advisor: <a href="https://cs.uwaterloo.ca/~m2nagapp/" target="_blank">Professor Meiyappan Nagappan</a></li>
      <li>Thesis Dissertation: Optimizing Automated Bug Localization for Practical Use [<a href="https://uwspace.uwaterloo.ca/items/55b444de-1a3a-4523-8617-48934e0fa670" target="_blank">Dissertation</a>]</li>
    </ul> 
  </p>
  <h2>Bangladesh University of Engineering and Technology</h2>
  <em>B.Sc.</em> in Computer Science and Engineering &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; Jul 2014 - Oct 2018
    <p>
      <ul>
  	    <li>Major: Software Engineering</li>
  	    <li>Advisor: <a href="https://rifatshahriyar.github.io/" target="_blank">Rifat Shahriyar</a></li>
	      <li>Thesis: An Empirical Study on the Growth of New Languages in Stack Overflow</li>
      </ul> 
    </p>
</div>

<div id="services">
  <h1>Services</h1>
  <div>
    <p>
      <ul>
        <li>Program Committee Member: Mining Software Repository 2025 Data and Tool Showcase Track</li>
        <li>Reviewer: 
          <ul>
            <li>IEEE Transactions on Software Engineering (TSE) - 2022, 2023, 2024 </li>
            <li>Empirical Software Engineering (EMSE) - 2024 </li>
            <li>ACM Transactions on Software Engineering and Methodology (TOSEM) - 2024 </li>
          </ul>
        </li>
        <li>Teaching:
          <ul>
            <li>TA for CSE 446 - Software Design and Architecture</li>
            <li>TA for CSE 348 - Introduction to Database Systems</li>
            <li>TA for CSE 230 - Introduction to Computers and Computer Systems</li>
          </ul>
        </li>
        <li>Mentored 1 UWaterloo CS undergraduate student on a research project.</li>
      </ul>
    </p>
  </div>
</div>


<div id="skills">
  <h1>Skills</h1>
  <div>
    <ul style="line-height: 1.8;">
      <li><b>Languages:</b> Python, Java, C++, C</li>
      <li><b>Frameworks:</b> PyTorch, TensorFlow, Sentence Transformers, ONNX, LiteLLM, Hugging Face</li>
      <li><b>Search & Retrieval:</b> Hybrid Search, BM25, Dense Retrieval, RRF, Learning to Rank, Bi-Encoder, Cross-Encoder, Hard Negative Mining, RAG</li>
      <li><b>Vector & Databases:</b> Milvus, Elasticsearch, Snowflake, PostgreSQL, MySQL</li>
      <li><b>Evaluation:</b> MAP, MRR, nDCG, Hit@K, Faithfulness, Context Relevance, Completeness</li>
      <li><b>Cloud & Infra:</b> AWS, SageMaker, Google BigQuery, Apache Spark, Docker</li>
      <li><b>Monitoring:</b> EvidentlyAI, Datadog, Prefect</li>
      <li><b>Tools:</b> Git, Linux, Shell Script</li>
    </ul>
  </div>
</div>

<div id="notes">
  <h1>Notes</h1>
  <div>
    <p>Occasional writing on things I have built or figured out the hard way.</p>
    <ul>
      {%- for post in site.posts -%}
        <li><a href="{{ post.url }}">{{ post.title }}</a> &ndash; {{ post.date | date: "%b %Y" }}</li>
      {%- endfor -%}
    </ul>
  </div>
</div>
