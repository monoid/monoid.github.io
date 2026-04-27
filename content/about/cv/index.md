---
title: "Ivan Boldyrev's Curriculum Vitae"
date: 2026-04-25T19:30:20+02:00
draft: false
ShowToc: true
TocOpen: true
---
 
## Contact
- **Contact:** [lispnik@gmail.com](mailto:lispnik@gmail.com)
- **LinkedIn:** [linkedin.com/in/ivan-boldyrev](https://www.linkedin.com/in/ivan-boldyrev)
- **GitHub:** [github.com/monoid](https://github.com/monoid/)
- **Location:** Barcelona, Catalonia, Spain

## Top Skills
- Languages: Rust . Python . C++
- Rust ecosystem: Tokio . axum . tonic . serde . Rkyv . WASM . libc
- Domains: Distributed systems . Performance engineering . Vector search . Cryptography (LWE, functional encryption, MPC) . Web crawling
- Other: WASM . System programming
- Open Source: WASM-SIMD implementation of [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) . Performance contributions to [Rkyv](https://github.com/rkyv/rkyv)

## Job Experience
 
### Qdrant Solutions GmbH
**Core Rust Developer**, remote contract\
*Sep 2025 - now*
- Optimized batch search operations by using cache more efficiently.
- Identified and fixed a memory safety issue (unsoundness) in the production Rust code.
- Analyzed performance of Qdrant under concurrent search-update workloads.
- Based on this analysis, designed a new concurrent architecture for Qdrant core using fine-grained locks and atomic operations.

### Stealth cryptography startup
**Rust Developer**, remote contract\
*Jun 2025 - Sep 2025 (3 months)*
- Co-researched an application of functional encryption for MPC (multi-party computation).
- Implemented function encryption schemes and an optimized LWE encryption algorithm.


### Cloudless Labs (Fluence Labs)
**Senior Rust Developer**, remote contract\
*May 2022 - Apr 2025 (2 years 11 months)*
- Contributed to development of AquaVM, a distributed language for Web3, with main focus on performance and correctness.  Built tooling and infrastructure, including a testing framework and a benchmark suite. Researched and implemented a data exchange format that significantly reduced memory size and execution time. Co-designed and implemented data signing scheme for distributed data.
- Researched and implemented internal representation and data serialization that reduced data size, memory size and execution time.
- Contributed to the development of a continuous hardware performance measurement tool
  ("proof of hardware capacity") that uses both Tokio and OS threads, including complex
  refactorings and writing code for low level memory manipulation (mmap et al). 
- Developed Rust Coding Standard for the company.

Stack: Rust, serde, Rkyv, Tokio, axum, tonic, prometheus-client, WASM, libc, cryptography crates.

### SEMrush
**Lead Software Engineer (Web Crawling)**, Saint-Petersburg, Russia\
*Jan 2020 - Oct 2021 (1 year 10 months)*
Owned a scheduler component of a web crawler and its dataset of 40T URLs.
- Improved the scheduler component that increased the crawling rate by 40% to 25B/day, while staying within politeness constraints.  It increased the index size, surpassing the planned target by almost 50%.
- Optimized both data pipeline speed and scheduler, reducing dataset cycle time from 3 to 2 days.
- Added tracking of pages’ changing frequency that allowed crawling rarely changing pages less frequently, reducing useless crawls.
Stack: Python, NumPy, C++ (incl. Boost.Python), Spark (PySpark) on Hadoop.
 
### Yandex
**Python and C++ Developer**, Novosibirsk, Russia\
*Feb 2014 - Nov 2019 (5 years 10 months)*
Was responsible for the <https://wordstat.yandex.ru>, Yandex's search engine
query search statistics tool.
- Worked on large data processing pipelines (YTsaurus MapReduce, Yandex in-house data processing cloud) for processing terabytes of daily data.
<!-- TODO developed -->
- Developed and maintained custom indices (short-living and long-living) from user search logs.
- Took full responsibility for the project, including porting it to new internal cloud infrastructure and adapting it to new requirements and data sources.
- Worked on a few data processing projects written in Java 8.
 
Stack: Python, C++, YTsaurus MapReduce, Yandex's in-house cloud.

### BARS Group
**Python Developer**, Novosibirsk, Russia\
*Dec 2013 - Feb 2014 (3 months)*
 
- Developed a web application for accountants as part of a medium-sized team.

Stack: Django for backend, JavaScript with Sencha ExtJS for frontend.
 
### Institute of Geology and Mineralogy of SB RAS
**Leading Software Engineer**, Novosibirsk, Russia\
*September 2000 - December 2013 (13 years 4 months)*
 
- Developing applied software for web cartography using JavaScript, OpenLayers, jQuery, Google Maps API, Geoserver, Java applets, Python/Django/Twisted, and PostGIS for PostgreSQL at the backend.
- Worked on satellite imagery processing using IDL (Interactive Data Language), including ENVI extensions.
 
## Education
 
**Novosibirsk State University (NSU)**, Novosibirsk, Russia\
BS, Computer Science and Applied Mathematics (1998 - 2002)
 
## Languages
- **Russian:** Native
- **English:** Professional Working
- **French:** Basic Proficiency
 
