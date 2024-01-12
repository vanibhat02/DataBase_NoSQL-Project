A Graph Database Powered Recommendation System


The goal of this project is to make use of Neo4j database to provide movie recommendations. This document gives the step by step instruction from software installation to running the query/program to generate the output.

I.  Neo4j Desktop Installation

•	Go to https://neo4j.com/download/
•	When you click on download it will prompt you to fill up few details Name, business email and company Name. Fill the details and click on ‘Download Desktop’
•	You will get .dmg file and activation key
•	Follow the simple UI driven clicks and you will be good to go
•	Launch the Neo4j application

II. Creating project and instance of database in Neo4j

•	Open neo4j application.
•	On the left hand side of the window, at the top, click on New>>Create new project to create new project. A default project with name Project 1 appears at the left hand side
•	Rename the project 
•	At the right hand side of the project created, click on ‘+Add’ then select ‘Local DBMS’ to create a DBMS 
•	Give a ‘Name’, a password which need to remember, select latest available version and click ‘Create’
•	It will take few minutes to download, install and create project. Wait for it!!
•	Now, you are all set

III. Installing plugins

•	Hover over the newly created DBMS and click on it. A window appears at the right side with details, plugins, and Upgrade.
•	Click on ‘Plugins’
•	Expand ‘APOC’ and install the plugin
•	Expand ‘Graph Data Science library’ and install the plugin
•	Close the window.
•	Hover over the newly created database and click on ‘start’
•	Wait until the DBMS is started. Once started it displays as ‘active’



IV. Importing data into Neo4j 

•	Hover over the DBMS you created and click on the 3 dots at the right hand side. Select Open folder >> Import. This opens up the import folder.
•	Drop the dataset into this folder

V. Start the Neo4j browser

•	Hover over the created DBMS and click on open >> Neo4j browser
•	Write/paste the below provided queries in the Neo4j$ and run it to get the results

VI. Import Dataset and define nodes and relationships

	•	About dataset 

		In Dataset	  In Neo4j	 Defined as
		userId	           User	          Node
		movieId	 	   movieId	  Property of User node
		rating	  	   RATED	          Edge from User to Movie
		movie_title         Movie	          Node
		director	           Director	  Node, 'DIRECTED' edge from director to movie
		genres	           Genre	          Node, 'IN_GENRE',edge from movie to Genre


•	Use the below code to import data and create above defined nodes and relationships

		LOAD CSV WITH HEADERS FROM 'FILE:/Data_05232022.csv' as row with row where row.userId is not null
		MERGE(u:User{Name:row.userId}) 
		MERGE(d:Director{Name:row.director})
		MERGE(m:Movie{Name:row.movie_title})
		MERGE (u) -[:RATED{rate:row.rating}]->(m)
		MERGE (d)-[:DIRECTED]->(m)
		FOREACH (Genre IN split(row.genres, '|') |
		MERGE (g:Genre {Name: Genre})
		MERGE (m)-[:IN_GENRE]->(g)
		)
		WITH u, collect(row.movie_title) as mov_agg, collect(row.genres) as gen_agg, collect(distinct row.director) as dir_agg,
 		collect(toInteger(row.movieId)) as movieid_agg
		SET u.Movies = mov_agg, u.Genres = gen_agg, u.Directors = dir_agg, u.MovieId = movieid_agg

•	View the graph that you created:
 		MATCH (n) RETURN (n)





VII. Use-cases and respective programs

1. Content based filtering
	  
Provide recommendation based on the history of the user without considering other users.

   Use-Case-1: Recommendation based on user’s genre preference
			This is a simple content based filtering recommendation without using any    formulas/functions. We are recommending 	
			movies similar to those genre the user has already watched except the movies that  he has already watched.                 


		MATCH (u:User {Name: "1"})-[r:RATED]->(m:Movie),
		(m)-[:IN_GENRE]->(g:Genre)<-[:IN_GENRE]-(rec:Movie)
		WHERE NOT EXISTS( (u)-[:RATED]->(rec) )
		WITH rec, [g.Name, COUNT(*)] AS scores
		RETURN rec.Name AS recommendation,
		COLLECT(scores) AS scoreComponents,
		REDUCE (s=0,x in COLLECT(scores) | s+x[1]) AS score
		ORDER BY score DESC


   Use-Case-2: Weighted recommendation based on user’s genre or director   preference
			Recommendation based on the movies genres and directors the users has already watched. We have used the weighted 
			technique where more weight has been assigned to genre

		MATCH (u:User {Name: "1"})-[r:RATED]->(m:Movie)-[:IN_GENRE]-(t)-[:IN_GENRE]-(rec:Movie)
		WHERE NOT EXISTS( (u)-[:RATED]->(rec) )
		WITH m, rec, COUNT(*) AS gs
		OPTIONAL MATCH (u:User {Name: "1"})-[r:RATED]->(m:Movie)-[d:DIRECTED]-(t)-[:DIRECTED]-(rec:Movie)
		WITH m, rec, gs, COUNT(d) AS ds
		RETURN m.Name, rec.Name AS recommendation, (5*gs)+(4*ds) AS score ORDER BY score DESC





    Use-Case-3: Jaccard similarity recommendation based on user’s genre or director  preference
			 Recommendation based on the movies genres/directors the users has already watched. We are using Jaccard based 
			 similarity index to provide recommendation. 
 
		MATCH (u:User {Name: "1"})-[r:RATED]->(m:Movie)-[:IN_GENRE|DIRECTED]-(t)-[:IN_GENRE|DIRECTED]-(other:Movie)
		WHERE NOT EXISTS( (u)-[:RATED]->(other))
		WITH m, other, COUNT(t) AS intersection, COLLECT(t.Name) AS i
		MATCH (m)-[:IN_GENRE|DIRECTED]-(mt)
		WITH m, other, intersection,i, COLLECT(mt.Name) AS s1
		MATCH (other)-[:IN_GENRE|DIRECTED]-(ot)
		WITH m,other,intersection,i, s1, COLLECT(ot.Name) AS s2
		WITH m,other,intersection,s1,s2
		WITH m,other,intersection,s1+[x IN s2 WHERE NOT x IN s1] AS union, s1, s2
		RETURN m.Name, other.Name,intersection,s1,s2,((1.0*intersection)/SIZE(union)) AS jaccard ORDER BY jaccard DESC


2. Collaborative filtering
	
     Provide recommendation to a user based on other similar users.

	Use-Case-1: User based collaborative filtering : Recommendation based on other similar users 
	        This is a simple collaborative filtering recommendation without using any formulas/functions. We are
                 recommending movies to a user based on other similar users 

		MATCH (u:User {Name: "1"})-[:RATED]->(:Movie)<-[:RATED]-(o:User)
		MATCH (o)-[:RATED]->(rec:Movie)
		WHERE NOT EXISTS( (u)-[:RATED]->(rec))
		RETURN DISTINCT rec.Name 
 


	Use-Case-2: Item based collaborative filtering : Recommending movies based on users average ratings and genre preference 

		For a particular user, what genres have a higher-than-average rating? Use this to score similar movies
 
		MATCH (u:User {Name: "1"})-[r:RATED]->(m:Movie)
		WITH u, avg(toInteger(r.rate) )AS mean
		MATCH (u)-[r:RATED]->(m:Movie)-[:IN_GENRE]->(g:Genre)
		WHERE (toInteger(r.rate) > mean)
		WITH u, g, COUNT(*) AS score
		MATCH (g)<-[:IN_GENRE]-(rec:Movie)
		WHERE NOT EXISTS((u)-[:RATED]->(rec))
		RETURN rec.Name AS recommendation,COLLECT(DISTINCT g.Name) AS genres, SUM(score) AS sscore
		ORDER BY sscore DESC
 
 
 	Use-Case-3: User based collaborative filtering : Cosine Similarity recommendation based on user ratings
 		
	         Recommending movies based on most similar user preferences to user 1 on user ratings by using cosine similarity
            	function.
 
	
 
		MATCH (p1:User{Name:'1'})-[x:RATED]->(movie)<-[x2:RATED]-(p2:User)
		WHERE p2 <> p1
		WITH p1, p2, collect(toFloat(x.rate)) AS p1Ratings, collect(toFloat(x2.rate)) AS p2Ratings
		WHERE size(p1Ratings) > 1
		MATCH (p2)-[:RATED]->(rec:Movie)
		WHERE NOT EXISTS ((p1)-[:RATED]->(rec))
		RETURN p1.Name AS User,p2.Name AS Similar_User,
		round(gds.similarity.cosine(p1Ratings, p2Ratings),3) AS Cosine_similarity, rec.Name AS Recommendations
		ORDER BY Cosine_similarity DESC





	Use-Case-4:User based collaborative filtering : KNN recommendation based on similar users movie preferences.

	         We are recommending movies to the user based on other similar users movie preferences. We are using KNN model to 
    	         generate recommendations.


CALL gds.graph.project(
    'myGraph',
    {
        Person: {
            label: 'User',
            properties: ['MovieId']
        }
    },
    '*'
);


CALL gds.knn.write.estimate('myGraph', {
  nodeProperties: ['MovieId'],
  writeRelationshipType: 'SIMILAR',
  writeProperty: 'score',
  topK: 1
})
YIELD nodeCount, bytesMin, bytesMax, requiredMemory


CALL gds.knn.stream('myGraph', {
    topK: 1,
    nodeProperties: ['MovieId'],
    // The following parameters are set to produce a deterministic result
    randomSeed: 1337,
    concurrency: 1,
    sampleRate: 1.0,
    deltaThreshold: 0.0
})
YIELD node1, node2, similarity
RETURN gds.util.asNode(node1).Name AS Person1, gds.util.asNode(node2).Name AS Person2, similarity
ORDER BY similarity DESCENDING, Person1, Person2


CALL gds.knn.stats('myGraph', {topK: 1, concurrency: 1, randomSeed: 42, nodeProperties: ['MovieId']})
YIELD nodesCompared, similarityPairs



CALL gds.knn.mutate('myGraph', {
    mutateRelationshipType: 'SIMILAR',
    mutateProperty: 'score',
    topK: 1,
    randomSeed: 42,
    concurrency: 1,
    nodeProperties: ['MovieId']
})
YIELD nodesCompared, relationshipsWritten




CALL gds.knn.write('myGraph', {
    writeRelationshipType: 'SIMILAR',
    writeProperty: 'score',
    topK: 1,
    randomSeed: 42,
    concurrency: 1,
    nodeProperties: ['MovieId']
})
YIELD nodesCompared, relationshipsWritten



MATCH (u)-[:SIMILAR]->(u2)
RETURN u,u2


MATCH (u:User{Name:'1'})-[:SIMILAR]->(u2)
RETURN u2.Movies


MATCH (u:User{Name:'1'})-[:SIMILAR]->(u2)
MATCH (u2)-[:RATED]->(rec:Movie)
WHERE NOT EXISTS((u)-[:RATED]->(rec))
RETURN rec


** END



