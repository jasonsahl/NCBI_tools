* Bash code
** TODO Are GCA files ever altered?
** rsync

#+BEGIN_SRC bash
#!/bin/bash

organism="bacteria/Acinetobacter_nosocomialis/" # or "bacteria/" for all bacteria
directory="./rsync_Acinetobacter_nosocomialis/" # directory will be created if doesn't already exist
files=".fna.gz"

rsync -iPrLtm -f="+ *"$files"" -f="+ */" -f="- *" ftp.ncbi.nlm.nih.gov::genomes/genbank/"$organism" "$directory" 

# -c, --checksum              skip based on checksum, not mod-time & size
# -i, --itemize-changes       output a change-summary for all updates
# -P, --partial, --progress   show progress and put partially downloaded files in a folder
# -r, --recursive             recurse into directories
# -L, --copy-links            transform symlink into referent file/dir
# -t, --times                 preserve modification times
# -m, --prune-empty-dirs      prune empty directory chains from file-list

#+END_SRC

* Everything below is probably not needed anymore, now that we have rsync.
** lftp
*** Sync with NCBI's ftp server

#+BEGIN_SRC sh :tangle ncbi_sync
# add time stamp to log file
lftp -c 'open -e "mirror -cf - --no-empty-dirs -I *assembly*.txt -P=5 --log=lftp_log.txt /genomes/genbank/bacteria/ ~/MGGen/ncbi_bacteria_mirror" ftp.ncbi.nlm.nih.gov'
#+END_SR

** wget
*** assembly summary for specific organism

#+BEGIN_SRC sh :tangle ncbi_get_organism
wget -O Acinetobacter_nosocomialis_summary.txt ftp://ftp.ncbi.nlm.nih.gov/genomes/genbank/bacteria/Acinetobacter_nosocomialis/assembly_summary.txt
awk -F "\t" '/Acinetobacter/ {print $20}' Acinetobacter_nosocomialis_summary.txt | \
sed -r 's|(ftp://ftp.ncbi.nlm.nih.gov/genomes/all/)(GCA_.+)|\1\2/\2_genomic.fna.gz|'>genome_urls.txt
wget -P ./genome_files/ --input genome_urls.txt
gunzip ./genome_files/*.gz
#+END_SRC
 
*** assembly summary for all bacteria

Since it downloads every summary as `assembly_summary.txt` in a separate directory named after the bacteria, we will have to copy and rename the files.  The files must remain in the structure that `lftp` puts them in in order to stay synced with NCBI.

#+BEGIN_SRC bash
mkdir bacteria_summaries && cd bacteria_summaries
wget -rl 0 -A "*assembly*.txt" -nH --cut-dirs=3 --no-remove-listing ftp://ftp.ncbi.nlm.nih.gov/genomes/genbank/bacteria
for subdir in *; do cp ./$subdir/assembly_summary.txt ./renamed/$subdir.txt; done;
#+END_SRC

- It has come to my attention that most organisms (~8K out of 12K) do NOT have a `assembly_summary.txt` file in the expected directory.  Running another test to find out where/if the equivalent file is located.
** Questions

- It is recommended to run the following script in a ~crontab~.  I don't really understand this script and was wondering if it's just as well to simply run ~lftp -c 'open -e "mirror -c -p --no-empty-dirs -I *assembly*.txt -P=5 --log=lftp_log.txt /genomes/genbank/bacteria/ /home/truthling/MGGen/ncbi_bacteria_mirror" ftp.ncbi.nlm.nih.gov'~ in a crontab?

#+BEGIN_SRC shell
#!/bin/bash
login="username"
pass="password"
host="server.feralhosting.com"
remote_dir='~/folder/you/want/to/copy'
local_dir="$HOME/lftp/"

base_name="$(basename "$0")"
lock_file="/tmp/$base_name.lock"
trap "rm -f $lock_file; exit 0" SIGINT SIGTERM
if [ -e "$lock_file" ]
then
    echo "$base_name is running already."
    exit
else
    touch "$lock_file"
    lftp -u $login,$pass $host << EOF
    set ftp:ssl-allow no
    set mirror:use-pget-n 5
    mirror -c -P5 --log="/var/log/$base_name.log" "$remote_dir" "$local_dir"
    quit
EOF
    rm -f "$lock_file"
    trap - SIGINT SIGTERM
    exit
fi
#+END_SRC
