git status

Your branch is ahead of 'origin/develop' by 102 commits.

git diff --numstat master.. | awk 'NF==3 {plus+=$1; minus+=$2} END {printf("+%d, -%d\n", plus, minus)}'

+12625, -27666
