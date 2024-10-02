#!/bin/bash
# This script updates the Docutils website safely

set -e  # Exit immediately if a command exits with a non-zero status
set -u  # Treat unset variables as an error

# Function to print feedback messages
print_feedback() {
    if [[ "${feedback:-1}" -eq 1 ]]; then
        echo "$1"
    fi
}

# Function to handle cleanup on exit
cleanup() {
    rm -rf "$lockdir" || true
    trap - EXIT
}
trap cleanup EXIT

# Project setup
svnurl="http://svn.code.sf.net/p/docutils/code/trunk"
htmlfilelist="$(pwd)/htmlfiles.lst"
basedir="$(pwd)/update-dir"

# Create the base directory if it doesn't exist
mkdir -p "$basedir"

project="docutils"
auxdir="$basedir/aux"
mkdir -p "$auxdir"
htdocsdest="$auxdir/htdocs"
mkdir -p "$htdocsdest"
snapshotdir="$auxdir/snapshots"
mkdir -p "$snapshotdir"

# Remote project setup
remoteproject="/home/project-web/docutils"
remotehtdocs="$remoteproject/htdocs"

pylib="$auxdir/lib/python"
lib="$pylib/$project"
lockdir="$auxdir/lock"

# Base URL for the project
baseurl="http://docutils.sourceforge.net"

export PYTHONPATH="$pylib:$lib:$lib/extras"
export PATH="$lib/tools:$PATH"

# Command-line options
tracking=0
unconditional=0
verbose=0
feedback=1

while getopts "ftuvq" opt; do
    case $opt in
        f) feedback=0;;
        t) tracking=1;;
        u) unconditional=1;;
        v) verbose=1;;
        q) verbose=0;;
        \?) exit 2;;
    esac
done
shift $((OPTIND - 1))

print_feedback 'Starting the document update process...'

if [[ "$tracking" -eq 1 ]] || [[ "$verbose" -eq 1 ]]; then
    set -o xtrace
fi

# Locking mechanism
if ! mkdir "$lockdir"; then
    echo "Cannot create lock directory at $lockdir. Ensure no other users are running this script."
    exit 1
fi
chmod 0770 "$lockdir"  # Ensure directory permissions are correct

# Update library
if [[ -e "$lib" ]]; then
    cd "$lib" || exit
    svn up --quiet
else
    mkdir -p "$pylib"
    cd "$pylib" || exit
    svn checkout "$svnurl/docutils"
fi

# Create snapshots
cd "$snapshotdir" || exit
for DIR in web docutils sandbox; do
    if [[ ! -d "$DIR" ]]; then
        svn checkout "$svnurl/$DIR"
    fi
done

# Create snapshots
tar -cz --exclude='.svn' -f "$project-snapshot.tgz" "$project"
tar -cz --exclude='.svn' -f "$project-sandbox-snapshot.tgz" sandbox

# Update htdocs
cd "$snapshotdir" || exit

copy_to_htdocsdest() {
    find "$@" -type d -name .svn -prune -o \( -type f -o -type l \) -print0 | \
        xargs -0 cp --no-dereference --update --parents \
            --target-directory="$htdocsdest"
}

# Update htdocs
copy_to_htdocsdest sandbox
(cd "$project" && copy_to_htdocsdest *)
(cd web && copy_to_htdocsdest * .[^.]*)

# Generate HTML files from .txt
cd "$htdocsdest/tools" || exit
find ../.. -name Makefile.docutils-update | while read -r makefile; do
    dir=$(dirname "$makefile")
    (cd "$dir" && make -f Makefile.docutils-update -s)
done

cd "$htdocsdest" || exit

# Create empty or old HTML files to force HTML generation
find documents -type f -name '*.txt' -print | while read -r txtfile; do
    dir=$(dirname "$txtfile")
    base=$(basename "$txtfile" .txt)
    htmlfile="$dir/$base.html"
    if [[ ! -e "$htmlfile" ]]; then
        print_feedback "Touching $htmlfile"
        touch -t 200001010101 "$htmlfile"
    fi
done

# Generate HTML from .txt files
for htmlfile in $(find . -name '*.html'); do
    dir=$(dirname "$htmlfile")
    base=$(basename "$htmlfile" .html)
    txtfile="$dir/$base.txt"
    [[ ! -e "$txtfile" ]] && txtfile="$dir/$base.rst"
    [[ ! -e "$txtfile" ]] && txtfile="$dir/$base"
    if [[ ! -e "$txtfile" ]]; then
        print_feedback "Warning: Input not found: $dir $base"
    else
        if [[ "$unconditional" -eq 1 ]] || [[ "$txtfile" -nt "$htmlfile" ]]; then
            if [[ "${base:0:4}" == "pep-" ]]; then
                print_feedback "$txtfile (PEP)"
                python "$lib/tools/rstpep2html.py" --config="$dir/docutils.conf" "$txtfile" "$htmlfile"
            else
                print_feedback "$txtfile"
                python "$lib/tools/rst2html5.py" --config="$dir/docutils.conf" "$txtfile" "$htmlfile"
            fi
        fi
    fi
done

# Generate sitemap XML for search engines
cd "$htdocsdest" || exit
if [[ -n "${haschanges:-}" ]]; then
    {
        echo '<?xml version="1.0" encoding="UTF-8"?>'
        echo '<urlset xmlns="http://www.google.com/schema/sitemap/0.84">'
        find . -name '.[^.]*' -prune -o -type d -printf '%p/\n' -o \( -type f -o -type l \) -print | while read -r i; do
            if [[ "$i" == "./" ]]; then
                i=index.html
                url="$baseurl/"
            elif [[ "$i" == "./sitemap" ]] || [[ "${i: -1}" == "/" && -f "${i}index.html" ]]; then
                continue
            else
                url="$baseurl${i:1}"
                url="${url// /%20}"
            fi
            lastmod=$(date --iso-8601=seconds -u -r "$i")
            lastmod="${lastmod::22}:00"
            if [[ "${i: -5}" == ".html" ]]; then
                priority=1.0
            elif [[ "${i: -4}" == ".txt" ]]; then
                priority=0.5
            else
                priority=0.2
            fi
            echo "<url><loc>$url</loc><lastmod>$lastmod</lastmod><priority>$priority</priority></url>"
        done
        echo '</urlset>'
    } > sitemap
    gzip -f sitemap
fi

# Push changes to remote server
cd "$htdocsdest" || exit
print_feedback "Syncing to SF"
rsync -e ssh -r -t ./ web.sourceforge.net:"$remotehtdocs"

print_feedback '...docutils-update done'
