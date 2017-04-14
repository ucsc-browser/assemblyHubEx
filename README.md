This repository has an example assembly hub, and outlines the steps
used to create the assembly hub from the original fasta file (Daphnia_pulex.allmasked.gz)
of the organism <em>Daphnia pulex</em> (<b>daph</b> used below). It also shows how GitHub is the kind
of server that allows byte-range requests needed to access
the binary indexed files that exist in hubs (provided files are under GitHub's file-size limit).
Another location where academic scientists can host files for free that support byte-range requests is the NSF funded site CyVerse https://de.cyverse.org/

Resource on assembly hubs:
http://genomewiki.ucsc.edu/index.php/Assembly_Hubs

This link provides info and an example hub to start with:
http://genome.ucsc.edu/goldenPath/help/hubQuickStartAssembly.html

Pulls an example hub
<pre>
wget -r --no-parent --reject "index.html*" -nH --cut-dirs=3 http://genome.ucsc.edu/goldenPath/help/examples/hubExamples/hubAssembly/plantAraTha1/
</pre>

It is best to immediately test if the hub loads (it should without any changes) from where you are hosting the files.
This is needed to be sure your server is allowing byte-range requests.

Once you know this example hub works, it can be edited, I suggest making a duplicate renamed copy of the files that you'll edit for your new hub.

1. First step change folder names
--
<pre>
mv plantAraTha1/ daph
mv araTha1/ daph
</pre>

2. Then change the hub.txt
----
Edit the hub.txt
<pre>
hub daph
shortLabel daph hub
longLabel Daph Hub
genomesFile genomes.txt
email genome-www@soe.ucsc.edu
descriptionUrl http://daphniagenomes.org/downloads
</pre>

3. Then change the genomes.txt 
---
Edit the genomes.txt
-The defaultPos will need to be further edited to have the correct chrom names.

<pre>
genome daph
trackDb daph/trackDb.txt
groups groups.txt
description Daph genome
twoBitPath daph/daph.2bit
organism
defaultPos chr1:1000000-2000000
orderKey 1000
scientificName Daphnia pulex
htmlPath localFile.html
</pre>

4. Then change the trackDb.txt 
--
Edit the trackDb.txt file

Remove everything put in a place holder track we'll create later

<pre>
track myTrack
longLabel My Track Name
shortLabel My Track
priority 10
visibility dense
color 0,0,0
bigDataUrl daph.bb
type bigBed 4
group map
</pre>

5. Remove all the files now unrelated.
------

In the location of the trackDb.txt (daph/ folder) remove all the other files

<pre>
rm a*
rm *.html
rm A*
rm -r bbi
</pre>

6. Build the 2bit file that will be the genome sequence displayed for the Assembly Hub
----
Now we'll build the 2bit file (daph.2bit described in genomes.txt) using the fasta.
faToTwoBit

Acquire your fasta file, source for this example:
http://genome.jgi.doe.gov/Dappu1/download/Daphnia_pulex.allmasked.gz

decompress the fasta
----
gzip -d Daphnia_pulex.allmasked.gz 

make the 2bit
Obtain utilities like faToTwoBit from:  http://hgdownload.soe.ucsc.edu/admin/exe/
--
<pre>
$ faToTwoBit Daphnia_pulex.allmasked daph.2bit
</pre>

(note there are options like -noMask        Ignore lower-case masking in fa file.)
NOTE: the utility faToTwoBit is available HERE: http://hgdownload.soe.ucsc.edu/admin/exe/ (as is the later findMotif tool) 

7. Create a chrom.sizes file that will be needed to build indexed annotation files like bigBeds or bigWigs.
----
Now we need to get the chrom info, http://genomewiki.ucsc.edu/index.php/Assembly_Hubs, shares more about building tracks for your hub which will require matching chrom names.

twoBitInfo can provide us the chrom.sizes file.  To make it pretty we sort it too.

twoBitInfo daph.2bit stdout | sort -k2rn > daph.chrom.sizes


This shows us that scaffold_1 4193030 is the largest item, we can set our default location in genomes.txt to something on this scaffold if we wish.

8. Update the correct name of chromosomes in genomes.txt 
-------Update genomes.txt 

<pre>
defaultPos chr1:1000000-2000000
--->
defaultPos scaffold_1:10000-20000
</pre>

There is also the chance before the creation of the 2bit file to change the naming strategy in the input fasta. 

NOTE: when you make changes to your hub, you will want to edit &udcTimeout=1  to ask the hub to look for changes. 

9. Build some annotation data (for example motif matches) by created a bed file and then a bigBed from that file.
---
Now the hub will load, and is only missing any annotation data. 
Some default tracks come for free (these are calculated on the fly in the browsing window: Rest. Enzym Short Match, Base Postion).

We'll build a quick bigBed file for the Hub 

Using the findMotif tool I'll output a bed file of all the matches of  AAAAAA 
<pre>
findMotif -motif=AAAAAA -strand=+ daph.2bit  > motif.bed
</pre>

<pre>
$ head motif.bed 
scaffold_1       80     86      1       1000    +
scaffold_1       253    259     2       1000    +
</pre>

Then so this will load in a hub, this bed track will be turned into a binary indexed bigBed

First this file must be sorted
 sort -k1,1 -k2,2n motif.bed >  motif.sorted.bed

To make this file with the executable bedToBigBed the chrom.sizes are needed so the index can be correctly used.

$ bedToBigBed  motif.sorted.bed daph.chrom.sizes daph.bb

10. Update trackDb.txt to make more sense with the new annotation file created
----
Now the daph/trackDb.txt will have a daph.bb to find when we load the hub:

Let's update the trackDb.txt to be more informative and set the display to full.

<pre>
track myTrack
longLabel This track shows a match on the sequence AAAAAA
shortLabel Motifs
priority 10
visibility pack
color 0,120,0
bigDataUrl daph.bb
type bigBed 5
group varRep
</pre>

(this 'group varRep' was defined in our ../groups.txt file as label Variation, it could be renamed).


This hub should now work and you can load a URL to the hub like this on the My Hubs page:
http://genome.ucsc.edu/cgi-bin/hgHubConnect

https://raw.githubusercontent.com/ucsc-browser/assemblyHubEx/master/Daphnia/hubExamples/hubAssembly/daph/hub.txt

=====
Now we can go a step further and turn on BLAT for this assembly hub.
=====
..these will only be active while the gfServer commands are running from where the 2bit file is located.


11. Set up the gfservers
See more here: http://genomewiki.ucsc.edu/index.php/Assembly_Hubs#Adding_BLAT_servers  

<pre>
gfServer start localhost 17777 -trans -mask daph.2bit &
gfServer start localhost 17779 -stepSize=5 daph.2bit &
</pre>

Get your server name, and check the status:
$ hostname -i   
132.249.245.79

Look for response (you can choose your own numbers):
 gfServer status 132.249.245.79 17777
 gfServer status 132.249.245.79 17779

12. Edit the genomes.txt of your assembly hub to point to where your gfServes are located (two lines blat and transBlat).

<pre>
...
scientificName Daphnia pulex
htmlPath localFile.html
blat 132.249.245.79 17779
transBlat 132.249.245.79 17777
</pre>

Now you can blat sequence:
<pre>
TCACATTTCCAAAGTTATTCCATATTCGTTTGTTTATATTTTGTTCGGC
GGACGTTTCCGCTATTACGGTTTCCATCATTACGACTCGCGGTCGTGCCT
CATTATGAAATCATTTGAGGGGGAAATAAGGTTCTCAGCAACCAGGGGTA
</pre>
Or possible amino acid sequence:
<pre>
VGVALGPGSESDAVVPDETLLLGIYIWTPSGHLQFTQFKAGGLAGTSSAY
</pre>
---
NOTE: that when you stop your gfServers the blat feature will end.



...More about Links and Hubs:
Each now hub added to the Browser gets a unique number (and it is different on genome-asia and genome-euro).
When you discover your hub's number you can use it to build a link to the Browser to immediately display the hub:
  http://genome.ucsc.edu/cgi-bin/hgTracks?db=hub_129603_daph&hubUrl=https://raw.githubusercontent.com/ucsc-browser/assemblyHubEx/master/Daphnia/hubExamples/hubAssembly/daph/hub.txt
  
 or without the hub_1234 you can use the genome name (?genome=(name you gave it)&hubUrl=location of files:
 
  http://genome.ucsc.edu/cgi-bin/hgTracks?genome=daph&hubUrl=https://raw.githubusercontent.com/ucsc-browser/assemblyHubEx/master/Daphnia/hubExamples/hubAssembly/daph/hub.txt

You can also use that number to build a link to other sections like the blat tool (useful if you have your blat servers running):
  http://genome.ucsc.edu/cgi-bin/hgBlat?db=hub_129603_daph&hubUrl=https://raw.githubusercontent.com/ucsc-browser/assemblyHubEx/master/Daphnia/hubExamples/hubAssembly/daph/hub.txt

In these cases, on the machine genome.ucsc.edu the URL https://raw.githubusercontent.com/ucsc-browser/assemblyHubEx/master/Daphnia/hubExamples/hubAssembly/daph/hub.txt is associated with hub_####_genome = hub_129603_daph

If you created a new hub, or had a different genome in the hub, the db=hub_##new##_genomeNew would have to be updated.

You can also build sessions to your hub, so that you can really have a detailed display happening.
http://genome.ucsc.edu/cgi-bin/hgTracks?hgS_doOtherUser=submit&hgS_otherUserName=brianlee&hgS_otherUserSessionName=hub_129603_daph
 
===
Pulling this repository to build a hub:
===

You can use this command if you have git installed:
<pre>
git clone https://github.com/ucsc-browser/assemblyHubEx.git /folders/exampleHub
</pre>

<h1>Using GBIB and this repository</h1>
--
If you are using a GBiB you can install git with this command:
<pre>
sudo apt install git
</pre>
You can also ssh into your GBiB to make pasting commands easier, where the password is "browser".
<pre>
ssh browser@localhost -p 1235
</pre>

Then you can clone this repository into the GBiB shared folders directory calling it exampleHub:
<pre>
sudo git clone https://github.com/ucsc-browser/assemblyHubEx.git /folders/exampleHub
</pre>
This put all the appropriate files in your GBiB shared folder. You should be able to load your hub with this URL 
http://127.0.0.1:1234/folders/exampleHub/Daphnia/hubExamples/hubAssembly/daph/hub.txt

Paste that address in the GBiB My Hubs section of the Track Hub Page:
http://127.0.0.1:1234/cgi-bin/hgHubConnect

