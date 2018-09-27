
<style>
.text_cell_render {
font-family: Times New Roman, serif;
}
</style>

<h1> Brooklyn OpenstreetMap Case Study</h1>

For my case study, I have chosen Brooklyn, NY as my area of choice. <br>

Link: https://www.openstreetmap.org/export#map=11/40.6400/-73.9359 <br>

<p> I will start by getting a count of the the tags that are present in the file. </p>


```python
import xml.etree.cElementTree as ET
import pprint
from collections import defaultdict

def count_tags(filename):
        # YOUR CODE HERE
        tags = {}
        for event,elem in ET.iterparse(filename):
            tag = elem.tag
            if tag not in tags.keys():
                tags[tag] = 1
            else:
                tags[tag]+=1
        return tags
        
        
def test():

    tags = count_tags('brooklyn.osm')
    pprint.pprint(tags)
    
    

if __name__ == "__main__":
    test()
```

    {'bounds': 1,
     'member': 15995,
     'meta': 1,
     'nd': 633323,
     'node': 453449,
     'note': 1,
     'osm': 1,
     'relation': 621,
     'tag': 501573,
     'way': 86459}
    

As you can see this is a large dataset with 15,995 members, 501,572 tags, and 86,459 ways.

<h2> Problems in the dataset</h2>

<p> There was an inconsistency in the value tag regarding street names. Some street names would have abbreviations like `Ave` or `E`, and others would have no abbreviations. To make it clear and consistent, I made all of the street names their unabbreivated version. This can be shown in the code and output below.</p>


```python

OSMFILE = 'brooklyn.osm'
street_type_re = re.compile(r'\b\S+\.?$', re.IGNORECASE)


expected = ["Street", "Avenue", "Boulevard", "Drive", "Court", "Place", "Square", "Lane", "Road", 
            "Trail", "Parkway", "Commons"]

# UPDATE THIS VARIABLE
mapping = { "St": "Street",
            "Ave" : "Avenue",
            "Rd" : "Road",
            "W" : "West",
            "E" : "East" 
            }


def audit_street_type(street_types, street_name):
    m = street_type_re.search(street_name)
    if m:
        street_type = m.group()
        if street_type not in expected:
            street_types[street_type].add(street_name)


def is_street_name(elem):
    return (elem.attrib['k'] == "addr:street")


def audit(osmfile):
    osm_file = open(osmfile, "r")
    street_types = defaultdict(set)
    for event, elem in ET.iterparse(osm_file, events=("start",)):

        if elem.tag == "node" or elem.tag == "way":
            for tag in elem.iter("tag"):
                if is_street_name(tag):
                    audit_street_type(street_types, tag.attrib['v'])
    osm_file.close()
    return street_types


def update_name(name, mapping):
    words = name.split()
    for w in range(len(words)):
        if words[w] in mapping:
            if words[w*1].lower() not in ['suite', 'ste.', 'ste']: 
                # For example, don't update 'Suite E' to 'Suite East'
                words[w] = mapping[words[w]]
                name = " ".join(words)
    return name


def test():
    st_types = audit(OSMFILE)
    pprint.pprint(dict(st_types))

    for st_type, ways in st_types.iteritems():
        for name in ways:
            better_name = update_name(name, mapping)
            print name, "=>", better_name
            


if __name__ == '__main__':
    test() #Data was inconsistent with street names, example Avenues would vary between using Avenue and the abbreviation Ave.
```

    {'1': set(['Rutland Rd #1']),
     'A': set(['Avenue A']),
     'Ave': set(['4th Ave', '7th Ave']),
     'B': set(['Avenue B']),
     'C': set(['Avenue C']),
     'D': set(['Avenue D']),
     'Extension': set(['Eastern Parkway Extension']),
     'F': set(['Avenue F']),
     'H': set(['Avenue H']),
     'Highway': set(['Kings Highway']),
     'I': set(['Avenue I']),
     'J': set(['Avenue J']),
     'K': set(['Avenue K']),
     'L': set(['Avenue L']),
     'M': set(['Avenue M']),
     'N': set(['Avenue N']),
     'North': set(['Paerdegat Avenue North']),
     'Plaza': set(['Brookdale Plaza', 'Newkirk Plaza']),
     'Rd': set(['Albemarle Rd']),
     'Remsen': set(['Remsen']),
     'Roadbed': set(['4th Avenue Southbound Roadbed',
                     'Kings Highway Westbound Roadbed',
                     'Ocean Parkway Southbound Roadbed']),
     'South': set(['Paerdegat Avenue South']),
     'Southwest': set(['Prospect Park Southwest']),
     'St': set(['E 92nd St', 'E 95th St']),
     'Terrace': set(['Albemarle Terrace', 'Kenmore Terrace']),
     'West': set(['Prospect Park West'])}
    4th Avenue Southbound Roadbed => 4th Avenue Southbound Roadbed
    Kings Highway Westbound Roadbed => Kings Highway Westbound Roadbed
    Ocean Parkway Southbound Roadbed => Ocean Parkway Southbound Roadbed
    Prospect Park West => Prospect Park West
    Albemarle Rd => Albemarle Road
    Kings Highway => Kings Highway
    Prospect Park Southwest => Prospect Park Southwest
    Rutland Rd #1 => Rutland Road #1
    Paerdegat Avenue North => Paerdegat Avenue North
    Remsen => Remsen
    Avenue A => Avenue A
    Avenue C => Avenue C
    Avenue B => Avenue B
    Avenue D => Avenue D
    Eastern Parkway Extension => Eastern Parkway Extension
    Avenue F => Avenue F
    Avenue I => Avenue I
    Avenue H => Avenue H
    Avenue K => Avenue K
    Avenue J => Avenue J
    Avenue M => Avenue M
    Avenue L => Avenue L
    Avenue N => Avenue N
    E 92nd St => East 92nd Street
    E 95th St => East 95th Street
    Paerdegat Avenue South => Paerdegat Avenue South
    Brookdale Plaza => Brookdale Plaza
    Newkirk Plaza => Newkirk Plaza
    Kenmore Terrace => Kenmore Terrace
    Albemarle Terrace => Albemarle Terrace
    7th Ave => 7th Avenue
    4th Ave => 4th Avenue
    

Now that the data is cleaned up I will move it on to a database where I can delve deeper into this dataset using SQL.

<h2> File sizes </h2>

brooklyn.osm - 93.507 MB <br>
nodes.csv - 42.06 MB <br>
nodes_tags.csv - 1.6 MB <br>
ways.csv - 5.7 MB <br>
ways_nodes.csv - 15.5 MB <br>
ways_tags.csv - 14.89 MB 

<h2> Checking City Tag</h2>

<p> When checking the city tag, I saw that Waterbury, was in one of them. Waterbury is a street in Brooklyn not a city.</p>


```SQL

SELECT tags.value, COUNT(*) as count
FROM (SELECT * FROM nodes_tags UNION ALL
       SELECT * FROM ways_tags) tags
WHERE tags.key LIKE %city
GROUP BY tags.value
ORDER BY count DESC;```

```ruby
Brooklyn,269
2,119
5,30 
10,5 
4,5
7,3
"New York",2
20,1
3,1
40,1
6,1
75,1
8,1
Waterbury,1
brooklyn,1```

 <p>I corrected it by changing the key value to `'street'` and changing the value to `'Waterbury Street'`</p>

<h2># of Amneties </h2>

```SQL
SELECT Count(*)
From nodes_tags
WHERE key = 'amenity'; ```

```ruby
704```

<h2>Most Popular Amnety</h2>

```SQL
SELECT value, COUNT(*) as num
 FROM nodes_tags
 WHERE key='amenity'
 GROUP BY value
 ORDER BY num DESC
 LIMIT 1;```

```RUBY
bicycle_parking,183```

Bicycle parking is the most popular amnety in this dataset which i find surprising. It takes up %25.99 of the amneties. 

<h2> # of places of worship</h2>

```SQL
SELECT COUNT(*)
FROM nodes_tags
	JOIN (SELECT DISTINCT(id) FROM nodes_tags WHERE value='place_of_worship') i
	ON nodes_tags.id=i.id
WHERE nodes_tags.key='religion';```

```ruby
119```

<h2>Top 10 Places of Worship</h2>

```SQL
SELECT nodes_tags.value, COUNT(*) as num
FROM nodes_tags 
    JOIN (SELECT DISTINCT(id) FROM nodes_tags WHERE value='place_of_worship') i
    ON nodes_tags.id=i.id
WHERE nodes_tags.key='religion'
GROUP BY nodes_tags.value
ORDER BY num DESC
LIMIT 10;```

```RUBY
christian|100
jewish|16
muslim|2
shinto|1```

<p>There are 4 types of religions with christianity being the most popular taking up %84.03 of its category. I also find there being a shinto shrine in Brooklyn interesting.</p>

<h2># of resturants</h2>

```SQL

SELECT COUNT(*) as num
FROM nodes_tags
	JOIN (SELECT DISTINCT(id) FROM nodes_tags WHERE value='restaurant') i
	ON nodes_tags.id=i.id
WHERE nodes_tags.key='cuisine';```

```ruby
47```

<h2>Top 10 Resturants</h2>

```SQL

SELECT nodes_tags.value, COUNT(*) as num
      FROM nodes_tags
      JOIN (SELECT DISTINCT(id) FROM nodes_tags WHERE value='restaurant') i
      ON nodes_tags.id=i.id
WHERE nodes_tags.key='cuisine'
GROUP BY nodes_tags.value
ORDER BY num DESC
limit 10;```

```rUBY
mexican|6
thai|4
pizza|3
burger|2
caribbean|2
chinese|2 
french|2 
italian|2 
vietnamese|2
american|1``` 

Mexican cuisine are the most popular. Another interesting thing to note is that there are a lot of resturants where only 1 of its type are in the set. To conclude I would like to say that I think this was an interesting dataset. There were things I was able to clean up and studying it led me to find new things like there being a shinto shrine in Brooklyn and get some statistics. It should be noted that this may not be Brooklyn as a whole but I do like that there has been data inputted by many people.

<h2> # of Unique Contributing Users</h2>

```SQL
SELECT COUNT(DISTINCT(e.uid))          
FROM (SELECT uid FROM nodes UNION ALL SELECT uid FROM ways) e;```

```ruby
381```

<h2> Feedback </h2>

<p>One way to improve this dataset would be to add more information to the streets. Information on if there's parking allowed on a certain street or not. It is very rough to find parking in Brooklyn as there are some areas you either cannot park in or can park in within a limited time frame. In order to implement there would need to be knowledge of cross streets to target exactly where a person would be able to park and link it to a unique ID. This would be beneficial to tourist and people wanting to find good parking within Brooklyn. A downside I can see is that it would add complexity to the data and there would need to be an update should parking laws change in the future.</p>


```python

```
