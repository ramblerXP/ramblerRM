# Backend Server Database Design

## Introduction

The backend will be indexing entities (places and historical sites, etc) from multiple data sources e.g: UNESCO, 
ASI, other governing bodies, our crawlers, or direct contributions from our users. We would need a robust design
for backend such that any changes in future coming from these data sources can be migrated to our exsiting database. 
We'll be using hexagonal design pattern for backend along with expand contract pattern for these changes.
The adaptor should be able to parse data from data sources and seed our universal DB. For each entity there might
be entries from different data sources, we'll create our own version based on user contribution, manual reviews,
etc but users can always switch to any of these data sources.

Each entity may have to undergo:

- Transalation pipeline: Data sources may provide entity data into different languages or Based on user's demand
  we might have to translate entity data in their language. So each entity must have a default language also which
  gets overwritten by User's requested language.
- Contribution Pipeline: All translated Entities must be stored and will be subject to review by contributors.
- Indexer Pipeline: Indexes all entities for frequent queries to reduce load on DB operations.

Contributions may be recieved in the form of comments, votes (up/down) etc.

## Resources

- https://developer.ibm.com/articles/an-introduction-to-uml/
- https://developer.ibm.com/articles/the-component-diagram/
- https://sparxsystems.com/resources/tutorials/uml/datamodel.html
