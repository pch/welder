#!/bin/sh

[ -n "$DEBUG" ] && set -x
set -eu

info() {
  printf "[ \033[00;34m..\033[0m ] %s" "$1"
}

success() {
  printf "\r\033[2K[ \033[00;32mOK\033[0m ] %s\n" "$1"
}

fail() {
  printf "\r\033[2K  [\033[0;31mFAIL\033[0m] %s\n" "$1"
  echo ''
  exit 1
}

help() {
  echo "Usage:"
  echo "    welder run <playbook> <ssh-url>"
  echo ""
  echo "Example:"
  echo "    welder run playbook.conf user@example.com"
  exit 1
}

if ! command -v rsync > /dev/null; then
  fail "Please install rsync first"
fi

[ $# -lt 3 ] && help

command=$1
case $command in
"-h" | "--help")
  help
  ;;
*)

  if [ "$1" != "run" ]; then
    help
  fi
  ;;
esac
shift 1

playbook_file=$1
ssh_url=$2
config_file="config.conf"

[ ! -f "$playbook_file" ] && fail "Playbook file $playbook_file not found"
[ ! -f "$config_file" ]   && fail "Config file $config_file not found"

WELDER_ROOT="$( cd -- "$(dirname "$0")" >/dev/null 2>&1 && pwd -P )"

output_dir="welder-cache"
mkdir -p $output_dir
cp -p "$WELDER_ROOT"/setup $output_dir

# Clean up playbook file (comments, empty lines)
grep -v -s -e '^#' -e '^$' "$playbook_file" > "$output_dir/$playbook_file"

# Clean up config: remove comments, empty lines and spaces around =
grep -v -s -e '^#' -e '^$' "$config_file" \
  | sed -e 's/[[:space:]]=[[:space:]]/=/' > "$output_dir/$config_file"

# Prepare config for *.template files, turn it into a sed command file
# First sed escapes variables so they can be used as... a sed pattern
# Source: https://stackoverflow.com/a/2705678
#
# Turns FOO="BAR" into s/FOO/BAR/g
sed -e 's/[]\/$*.^[]/\\&/g'  \
  -e 's/="/=/;s/"$//' \
  -e 's/^/s\/{{ /' \
  -e 's/=/ }}\//' \
  -e 's/$/\/g/' "$output_dir/$config_file" > "$output_dir/templates-$config_file"

# Go through all modules listed in $playbook_file, copy them to $output_dir
while read -r module; do
  cp -R "$module" "$output_dir"
done < "$output_dir/$playbook_file"

# Interpolate templates
find $output_dir -type f -name "*.template" | while IFS= read -r template; do
  template_output=${template%.*}
  sed -f "$output_dir/templates-$config_file" "$template" > "$template_output"
  rm "$template"
done

# Copy files to the server and run everything
rsync -a --delete --ignore-times --quiet "$output_dir"/ "$ssh_url":"$output_dir"
ssh -t "$ssh_url" "cd $output_dir && ./setup $playbook_file && cd .. && rm -r $output_dir"

success "All done!"
