http://genomewiki.ucsc.edu/index.php/Assembly_Hubs
Resource on assembly hubs

Provides info and a link to an example hub to start with:
http://genome.ucsc.edu/goldenPath/help/hubQuickStartAssembly.html

Pulls an example hub
wget -r --no-parent --reject "index.html*" -nH --cut-dirs=3 http://genome.ucsc.edu/goldenPath/help/examples/hubExamples/hubAssembly/plantAraTha1/

I suggest immediately testing the hub loads (it should without any changes) from where you are hosting the files.
This is needed to be sure your server is allowing byte-range requests.

Once you know this example hub works, it can be edited, I suggest making a duplicate renamed copy of the files that you'll edit for your new hub.

1. First step
--
changed name of folders
mv plantAraTha1/ daph
mv araTha1/ daph

2. Then changed the hub.txt
----
Edit the hub.txt

hub daph
shortLabel daph hub
longLabel Daph Hub
genomesFile genomes.txt
email genome-www@soe.ucsc.edu
descriptionUrl http://daphniagenomes.org/downloads

3. Then change the genomes.txt 
---
Edit the genomes.txt
-The defaultPos will need to be futher edited to have the correct chrom names.

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

4. Then change the trackDb.txt 
--
Edit the trackDb.txt file

Remove everything put in a place holder track we'll create later

track myTrack
longLabel My Track Name
shortLabel My Track
priority 10
visibility dense
color 0,0,0
bigDataUrl daph.bb
type bigBed 4
group map


5. Remove all the files now unrelated.
------

In the location of the trackDb.txt (daph/ folder) remove all the other files

rm a*
rm *.html
rm A*
rm -r bbi


6. Build the 2bit file that will be the genome sequence displayed for the Assembly Hub
----
Now we'll build the 2bit file (daph.2bit described in genomes.txt) using the fasta.
faToTwoBit

Not possible, required a login to download:
wget http://genome.jgi.doe.gov/Dappu1/download/Daphnia_pulex.allmasked.gz

decompress the fasta
----
gzip -d Daphnia_pulex.allmasked.gz 

make the 2bit
--
$ faToTwoBit Daphnia_pulex.allmasked daph.2bit

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

defaultPos chr1:1000000-2000000
--->
defaultPos scaffold_1:10000-20000


There is also the chance before the creation of the 2bit file to change the naming strategy in the input fasta. 

NOTE: when you make changes to your hub, you will want to edit &udcTimeout=1  to ask the hub to look for changes. 

9. Build some annotation data (for example motif matches) by created a bed file and then a bigBed from that file.
---
Now the hub will load, and is only missing any annotation data. 
Some default tracks come for free (these are calculated on the fly in the browsing window: Rest. Enzym Short Match, Base Postion).

We'll build a quick bigBed file for the Hub 

Using the findMotif tool I'll output a bed file of all the matches of  AAAAAA 
findMotif -motif=AAAAAA -strand=+ daph.2bit  > motif.bed

$ head motif.bed 
scaffold_1       80     86      1       1000    +
scaffold_1       253    259     2       1000    +

Then so this will load in a hub, this bed track will be turned into a binary indexed bigBed

First this file must be sorted
 sort -k1,1 -k2,2n motif.bed >  motif.sorted.bed

To make this file with the executable bedToBigBed the chrom.sizes are needed so the index can be correctly used.

$ bedToBigBed  motif.sorted.bed daph.chrom.sizes daph.bb

10. Update trackDb.txt to make more sense with the new annotation file created
----
Now the daph/trackDb.txt will have a daph.bb to find when we load the hub:

Let's update the trackDb.txt to be more informative and set the display to full.

track myTrack
longLabel This track shows a match on the sequence AAAAAA
shortLabel Motifs
priority 10
visibility pack
color 0,120,0
bigDataUrl daph.bb
type bigBed 5
group varRep

(this 'group varRep' was defined in our ../groups.txt file as label Variation, it could be renamed).


This hub should now work and you can load a URL to the hub like this on the My Hubs page:
http://genome.ucsc.edu/cgi-bin/hgHubConnect

http://hgwdev.cse.ucsc.edu/~brianlee/hubTestingAssembly/Daphnia/hubExamples/hubAssembly/daph/hub.txt

=====
Now we can go a step further and turn on BLAT for this assembly hub.
=====
..these will only be active while the gfServer commands are running from where the 2bit file is located.


11. Set up the gfservers
See more here: http://genomewiki.ucsc.edu/index.php/Assembly_Hubs#Adding_BLAT_servers  

gfServer start localhost 17777 -trans -mask daph.2bit &
gfServer start localhost 17779 -stepSize=5 daph.2bit &

Get your server name, and check the status:
$ hostname -i   
132.249.245.79

Look for response (you can choose your own numbers):
 gfServer status 132.249.245.79 17777
 gfServer status 132.249.245.79 17779

12. Edit the genomes.txt of your assembly hub to point to where your gfServes are located (two lines blat and transBlat).

...
scientificName Daphnia pulex
htmlPath localFile.html
blat 132.249.245.79 17779
transBlat 132.249.245.79 17777

Now you can blat sequence:
TCACATTTCCAAAGTTATTCCATATTCGTTTGTTTATATTTTGTTCGGC
GGACGTTTCCGCTATTACGGTTTCCATCATTACGACTCGCGGTCGTGCCT
CATTATGAAATCATTTGAGGGGGAAATAAGGTTCTCAGCAACCAGGGGTA

Or possible amino acid sequence:
VGVALGPGSESDAVVPDETLLLGIYIWTPSGHLQFTQFKAGGLAGTSSAY

---
NOTE: that when you stop your gfServers the blat feature will end.



...More about Links and Hubs:
Each now hub added to the Browser gets a unique number (and it is different on genome-asia and genome-euro).
When you discover your hub's number you can use it to build a link to the Browser to immediately display the hub:
  http://genome.ucsc.edu/cgi-bin/hgTracks?db=hub_128011_daph&hubUrl=http://hgwdev.cse.ucsc.edu/~brianlee/hubTestingAssembly/Daphnia/hubExamples/hubAssembly/daph/hub.txt

You can also use that number to build a link to other sections like the blat tool:
  http://genome.ucsc.edu/cgi-bin/hgBlat?db=hub_128011_daph&hubUrl=http://hgwdev.cse.ucsc.edu/~brianlee/hubTestingAssembly/Daphnia/hubExamples/hubAssembly/daph/hub.txt

In these cases, on the machine genome.ucsc.edu the URL http://hgwdev.cse.ucsc.edu/~brianlee/hubTestingAssembly/Daphnia/hubExamples/hubAssembly/daph/hub.txt is associated with hub_####_genome = hub_128011_daph

If you created a new hub, or had a different genome in the hub, the db=hub_##new##_genomeNew would have to be updated.

You can also build sessions to your hub, so that you can really have a detailed display happening.
http://genome.ucsc.edu/cgi-bin/hgTracks?hgS_doOtherUser=submit&hgS_otherUserName=brianlee&hgS_otherUserSessionName=hub_128011_daph
 
