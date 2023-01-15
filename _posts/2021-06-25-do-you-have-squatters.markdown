---
layout: post
title: "Do you have Squatters?"
strapline: "Unwanted house guests on the Internet ..."
date: 2021-06-25 
published: true
lastmod: 2021-06-25
changefreq: monthly
priority: 0.5
categories: howto dns
excerpt_separator: <!--excerpt-->
---

[Typosquatting](https://en.wikipedia.org/wiki/Typosquatting) <sup>[1](#1)</sup> is the Internets version of occupying empty buildings, without permission "[Squatting](https://en.wikipedia.org/wiki/Squatting)"<sup>[2](#2)</sup>. 

In this article, we describe a process that can be conducted to determine whether or not you have squatters on your branded domains. It is a manual process and covers the basic forms of typo and TLD squatting. It is intended to be carried out as a "[Proof of Concept](https://en.wikipedia.org/wiki/Proof_of_concept)" <sup>[3](#3)</sup> to ascertain if you have squatters and whether a more rigorous, automated process would be of value in your broader risk management process. 

The output of this process is a spreadsheet, from which you will be able to gain some basic statistics on "potential" squatting activity and a starting point for further investigatory work.

<!--excerpt-->

&nbsp;
## Methodology

You will require:

1. A domain name [domain]
2. A list of common spelling mistakes of the domain name
3. A list of TLD/ccTLD
4. Linux/OSX terminal and an Internet connection

We will use the Linux/OSX terminal to create and combine the typo and TLD lists into a master list of candidate domain names, then we will use the [dig](https://en.wikipedia.org/wiki/Dig_(command)) (domain information groper) <sup>[4](#4),[5](#5)</sup> utility to execute dig [Start of Authority](http://searchnetworking.techtarget.com/definition/start-of-authority-record) (SOA) <sup>[6](#6)</sup> queries to determine if the candidate domain has been registered and finish by using grep to gather all results and create a findings spreadsheet. 

### Typos

To create a list of possible typographical mistakes for a given domain name we will use the [Damerau–Levenshtein distance](https://en.wikipedia.org/wiki/Damerau%E2%80%93Levenshtein_distance) (DL) <sup>[7](#7)</sup> which is the the minimum number of operations; insertions, deletions or substitutions of a single character, and/or the transposition of two adjacent characters required to change one word into another.

We will use a Damerau–Levenshtein distance of 1, (a single edit) and the UTF8 domain name character set! for example:


<table>
<colgroup>
<col width="5%"/>
<col width="10%"/>
<col width="20%"/>
</colgroup>
<thead>
<tr class="header">
<th></th>
<th></th>
<th></th>
</tr>
</thead>
<tbody>

<tr>
<td></td>
<td>string</td>
<td>domain</td>
</tr>

<tr>
<td></td>
<td>charset</td>
<td>abcdefghijklmnopqrstuvwxyz0123456789-</td>
</tr>

<tr>
<td></td>
<td>insertion</td>
<td>adomain daomain domaain ...</td>
</tr>

<tr>
<td></td>
<td>deletion</td>
<td> omain dmain doain ...</td>
</tr>

<tr>
<td></td>
<td>substitution</td>
<td>aomain bomain comain ...</td>
</tr>

<tr>
<td></td>
<td>transposition</td>
<td>odmain dmoain doamin ... (adjacent character transposition)</td>
</tr>

</tbody>
</table>
&nbsp;

To create a file "[DOMAIN]-candidate-domains.txt" containing a list of DL 1 typo's, cut&paste the following into a terminal, replacing "domain" with the domain name of your choosing, without a TLD/ccTLD appended to the end! For example: if the domain name you wish to test is "google.com", replace "domain" with "google"

#### insertion  

	DOMAIN="domain"; \
	OUTPUT="$DOMAIN-candidate-domains.csv"; \
	charset="abcdefghijklmnopqrstuvwxyz0123456789-"; \
	type="i"; \
	read -a CHARS <<<"$(echo $charset | sed 's/./& /g')"; \
	for (( x=0; x<=${#DOMAIN}; x++ )); do  \
	START=${DOMAIN[@]:0:$x} ; \
	END=${DOMAIN[@]:$x:${#DOMAIN}} ; \
	for CHAR in "${CHARS[@]}"; do \
	echo "$START$CHAR$END,$type" |tee -a $OUTPUT ; \
	done; \
	done

#### deletion

	DOMAIN="domain"; \
	OUTPUT="$DOMAIN-candidate-domains.csv"; \
	type="d"; \
	read -a CHARS <<<"$(echo $DOMAIN | sed 's/./& /g')"; \
	for (( x=0; x<=${#DOMAIN}-1; x++ )); do  \
	s=""; \
	for (( y=0; y<=${#DOMAIN}-1; y++ )); do  \
	if [ $x != $y ]; then \
	s=$s${CHARS[$y]} ; \
	fi; \
	done; \
	echo "$s,$type" |tee -a $OUTPUT ; \
	done

#### substitution

	DOMAIN="domain"; \
	OUTPUT="$DOMAIN-candidate-domains.csv"; \
	charset="abcdefghijklmnopqrstuvwxyz0123456789-"; \
	type="s"; \
	read -a CHARS <<<"$(echo $charset | sed 's/./& /g')"; \
	for (( x=0; x<=${#DOMAIN}-1; x++ )); do  \
	START=${DOMAIN[@]:0:$x} ; \
	END=${DOMAIN[@]:$x+1:${#DOMAIN}} ; \
	for CHAR in "${CHARS[@]}"; do \
	echo "$START$CHAR$END,$type" |tee -a $OUTPUT ; \
	done; \
	done

#### transposition

	DOMAIN="domain"; \
	OUTPUT="$DOMAIN-candidate-domains.csv"; \
	type="t"; \
	for (( x=0; x<=${#DOMAIN}-2; x++ )); do  \
	START=${DOMAIN[@]:0:$x} ; \
	MID=${DOMAIN[@]:$x+1:1}${DOMAIN[@]:$x:1} ; \
	END=${DOMAIN[@]:$x+2:${#DOMAIN}} ; \
	echo "$START$MID$END,$type" |tee -a $OUTPUT ; \
	done

Open the "[DOMAIN]-candidate-domains.txt" in a text editor and sort/unique the contents. Remove any candidate domain names that start or end with a hyphen they are invalid. For a domain name of "domain", you should have a file containing 477 variations of "domain" from 0domain to zomain.


### TLD/ccTLD

You have several choices for creating a list of TLD's, “[?????]-tlds.txt”. However please be cognisant of the consequences; If you have 477 variations and 10 TLD's you will be executing 4770 dig queries and creating 4770 files. 

* TLD's by Popularity 

	W3Techs Web Technology Surveys <sup>[8](#8)</sup> maintains a "Usage of top level domains for websites" list
	
	[https://w3techs.com/technologies/overview/top_level_domain/all](https://w3techs.com/technologies/overview/top_level_domain/all)
	
	top_10_tlds.txt

	.com .ru .org .net .de .jp .uk .br .it .pl

* TLD's by Abuse 

	Spamhaus <sup>[9](#9)</sup> maintains many interesting list on spamming activity, the TLDs with the worst reputations for spam operations are: 
	
	[https://www.spamhaus.org/statistics/tlds/](https://www.spamhaus.org/statistics/tlds/)

	abused-tlds.txt

	.biz .click .cricket .gdn .gq .ml .party .review .study .top


### DIG 

Once you have your "[DOMAIN]-candidate-domains.txt" and "abused-tlds.txt" files you can cut&paste the following into the terminal. It will loop through both files, executing a dig SOA query for each candidate domain and piping the results into files. 

Note: it will take "some time"! You can CTRL-C at any time to stop it and re-cut&paste later and it will pick-up from where it left off.

	DOMAIN="domain"; \
	if [ ! -d $DOMAIN ]; then mkdir $DOMAIN; fi; \
	while read TLD; do \
	while IFS=, read domain type; do \
	CMD="dig @8.8.4.4 $domain$TLD SOA"; \
	OUTPUT=$(echo $CMD |sed 's/ /_/g'); \
	if [ ! -f $DOMAIN/$OUTPUT-$type.txt ]; then \
	echo "$CMD >$DOMAIN/$OUTPUT-$type.txt"; \
	$CMD >$DOMAIN/$OUTPUT-$type.txt; \
	fi; \
	done < $DOMAIN-candidate-domains.csv; \
	done < abused-tlds.txt

### Results

To create a "$DOMAIN-domains.csv" file cut&paste the following into the terminal. It will grep each file in the $DOMAIN directory, piping the results into the .csv file. It will also attempt to determine the TLD of the domain and domain of the SOA's Name Server for ease of sorting and categorising.

	DOMAIN="domain"; \
	OUTPUT="$DOMAIN-domains.csv"; \
	if [ -f $OUTPUT ]; then mv -f $OUTPUT $OUTPUT.bak; fi; \
	TAB=$'\t' ; \
	echo "domain,ttl,net,type,ns,email,serial,refresh,retry,expire,minimum,dl,tld,soadomain" |tee -a $OUTPUT ; \
	for FILE in $DOMAIN/dig*.txt; do \
	dl=$(echo "$FILE" |cut -d "_" -f 4 |sed "s/SOA-//g" |sed "s/.txt//g"); \
	domain=$(echo "$FILE" |cut -d "_" -f 3); \
	count=$(echo $domain |grep -o "\." |wc -l) ; \
	if [ $count == 1 ]; then \
	tld=$(echo $domain |cut -d "." -f 2); \
	elif [ $count == 2 ]; then \
	tld=$(echo $domain |cut -d "." -f 2,3); \
	fi; \
	if grep -q "^$domain" $FILE; then \
	LINE=$(grep "^$domain" $FILE |sed "s/${TAB}/ /g" |sed "s/  / /g" |sed "s/ /,/g" |tr '[:upper:]' '[:lower:]'); \
	soa=$(echo $LINE |cut -d , -f 5); \
	COUNT=$(echo $soa |grep -o "\." |wc -l) ; \
	if [ $COUNT == 2 ]; then \
	SOADOMAIN=$soa; \
	elif [ $COUNT == 3 ]; then \
	SOADOMAIN=$(echo $soa |cut -d "." -f 2,3); \
	else \
	SOADOMAIN=$(echo $soa |cut -d "." -f 2,3,4); \
	fi; \
	echo "$LINE,$dl,$tld,$SOADOMAIN" |tee -a $OUTPUT ; \
	else \
	echo "$FILE" |cut -d "_" -f 3 |tee -a $OUTPUT ; \
	fi; \
	done

&nbsp;
## Findings

The screenshot below is of the "$DOMAIN-domains.csv" spreadsheet for the "domain" domain and using the top 10 most popular TLD's and the top 10 most abused TLD's with the ttl, net, serial, refresh, retry, expire and minimum columns remove and sorted by the soadomain column. 

{% include image.html url="/assets/images/do-you-have-squatters-screenshot.png" description="findings spreadsheet"%}

The spreadsheet contains 9540 entries and has identified 707 potential domain squatters that will require individual investigation to determine if they are true Typosquatting domains or not. However, some general observations can be made on the spreadsheet as a whole:

* Domain names that have not been registered, 8833
* Potential domain squatters that require further investigation, 707
* Domains that have been registered by [domain parking](https://en.wikipedia.org/wiki/Domain_parking) <sup>[10](#10)</sup> companies; domaincontrol.com, parkingcrew.net, sedoparking.com and bodis.com etc.
* Damerau–Levenshtein statistics; insertion 405, deletion 39, substitution 240 and transposition 21
* TLD's statistics; com 224, net 115, de 98, ru 77, org 61 etc.
* SOA domain statistics; domaincontrol.com 59, uniregistrymarket.link 42, sedoparking.com 29, bodis.com 20, freenom.com 14 etc.

&nbsp;
## Conclusion

These terminal snippets are quite rudimentary and could be easily translated into a more functional programming language to improve the process and to make it more robust and repeatable. However, we hope that as a Proof of Concept for finding potential domain name squatters you found it to be of some value and worth undertaking.

&nbsp; &nbsp;
### Notes
1. <a name="1"></a>Typosquatting, also called URL hijacking, a sting site, or a fake URL ... 
"[Typosquatting](https://en.wikipedia.org/wiki/Typosquatting)" [Wikimedia Foundation, Inc.](https://www.wikimediafoundation.org/), 23 May 2017, accessed 1 Jun 2017 

2. <a name="2"></a>Squatting is the action of occupying an abandoned or unoccupied area of land or a building ...
"[Squatting](https://en.wikipedia.org/wiki/Squatting)" [Wikimedia Foundation, Inc.](https://www.wikimediafoundation.org/), 11 June 2017, accessed 1 Jun 2017 

3. <a name="3"></a>Proof of Concept (PoC) is a realization of a certain method or idea in order to demonstrate its feasibility ...
"[Proof of concept](https://en.wikipedia.org/wiki/Proof_of_concept)" 
[Wikimedia Foundation, Inc.](https://www.wikimediafoundation.org/), 11 May 2017, accessed 1 Jun 2017 

4. <a name="4"></a>dig (domain information groper) is a network administration command-line tool for querying Domain Name System (DNS) servers ... 
"[dig (command)](https://en.wikipedia.org/wiki/Dig_(command))" [Wikimedia Foundation, Inc.](https://www.wikimediafoundation.org/), 7 April 2017, accessed 1 Jun 2017 

5. <a name="5"></a>Heinlein, Paul. "[DiG HOWTO - How to use dig to query DNS name servers.](https://www.madboa.com/geek/dig/)" [madboa.com](https://www.madboa.com/), 11 May 2006, accessed 1 Jun 2017

6. <a name="6"></a>A start of authority (SOA) record is information stored in a domain name system (DNS) zone about that zone and about other DNS records ...
Rouse, Margaret. "[Start of Authority record](http://searchnetworking.techtarget.com/definition/start-of-authority-record)" [SearchNetworking](http://searchnetworking.techtarget.com/), April 2007, accessed 1 Jun 2017

7. <a name="7"></a>In information theory and computer science, the Damerau–Levenshtein distance (named after Frederick J. Damerau and Vladimir I. Levenshtein) is a string metric for measuring the edit distance between two sequences ...
"[Damerau–Levenshtein distance](https://en.wikipedia.org/wiki/Damerau%E2%80%93Levenshtein_distance)"
[Wikimedia Foundation, Inc.](https://www.wikimediafoundation.org/), 17 April 2017, accessed 1 Jun 2017 

8. <a name="8"></a>“[Usage of top level domains for websites](https://w3techs.com/technologies/overview/top_level_domain/all)” 
[W3Techs Web Technology Surveys](https://w3techs.com), 2017, accessed 1 Jun 2017 

9. <a name="9"></a> “[The 10 Most Abused Top Level Domains](https://www.spamhaus.org/statistics/tlds/)” 
[The Spamhaus Project Ltd](https://www.spamhaus.org), 2017, accessed 1 Jun 2017 

10. <a name="10"></a>Domain parking refers to the registration of an internet domain name without that domain being associated with any services such as e-mail or a website ...
"[Domain parking](https://en.wikipedia.org/wiki/Domain_parking)" [Wikimedia Foundation, Inc.](https://www.wikimediafoundation.org/), 23 May 2017, accessed 1 Jun 2017




