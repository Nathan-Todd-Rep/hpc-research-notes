# Inkly — Project Ideas

## Job Success Prediction System

- ML model that tries to predict whether a job will fail before it runs.
- Includes an explanation of why the job would fail.
- Work has already been done in this area.
- Two approaches: one easier (hooking up to Ollama), one harder.

## Gaussian Chemistry AI Data Repository

- Filter out data and find anything related to Galceon chemistry.
- Scrape website data for examples (e.g. Harvard datasets).
- Have the AI retrieve info from the database.
- Ollama can fetch from the internet but doesn't always work reliably.
- Scrape for keywords like "Galceon" in messages.


- Scraper could be in a seperate repository
- break it apart in seperate step
- grab from website form to json
- second step makes it pretty
- scrape stackoverflow (Could be difficult) find out if there is a special tag for gaussian.
- claude code parsing website

- Find a way to evaluate answers (look for check marks)
 - Bioframatics stack exchange (add scraping capability).
 - defer to Ollama to grab gaussian information if the user is talking about gaussian
 - check existing Ollama summarization.
 
 - Conda for running dependencies on the HPC
 - Virtual environment could work

- Revisit summarization storage
- 
## User Evaluation

- Very difficult to implement well.
- Find out Ollama model we are using, (one of the llamas)
- if we get it working try bioframatics



- Create a module where I just select the parameters for example now we're scraping gaussian but tomorrow we might be scraping bioframatics, a user could type a keyword or website and we could scrape from there
- consult with Andrew to find out what common software keywords are being used in the hpc 
- start our scraping, save it somewhere
- ingest all of that data into the inkly assistant
- make a metric to measure the quality of the responses
- Making it modular

	scraping previous slurm submissions. ask andrew for the anonymized data.
	anything I can learn about rag systems that could be the path we take to integrate the scraper with inkly.

	keep it as a command line tool for programmers to use

	We would scrape the submissions for users using gaussian look for error codes and what results they were getting, could be an additional data source. We could learn from pervious mistakes.
	
try taking passages and distill this knowledge into one paragraph using ollama 
Potential to get keys to a better ai model.

---

## Related Notes

- [[Work Session 1]] — first session, started the Gaussian scraper
- [[Gaussian Scraper Final Thoughts]] — progress on the Gaussian data project
- [[Inkly Codebase Notes Start]] — understanding the codebase before building
