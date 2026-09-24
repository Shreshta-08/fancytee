# fancytee
fancytee.js
# Project Description
FancyTee is a Node.js command based on the Linux tee command. It takes input, displays it on the screen, saves it to a file, and counts the number of lines written.
# AI Reflection
I asked AI to help me understand tee, write my Node.js command, and find possible errors and edge cases.
AI helped me find a problem with no input, but I tested the code myself and fixed the issue.

# Code
const fs = require('fs');
const path = require('path');

// 1. Check arguments
if (process.argv.length < 3) {
  console.error(`Usage: node ${path.basename(process.argv[1])} <filename> [append|overwrite]`);
  process.exit(1);
}

// 2. Read filename and optional mode
const filename = process.argv[2];
const mode = process.argv[3] || 'append';

// Check that the mode is valid
if (mode !== 'append' && mode !== 'overwrite') {
  console.error('Error: Mode must be "append" or "overwrite".');
  process.exit(1);
}

// 3. Set up variables
let lineCount = 0;
let lastChunkEndedWithNewline = false;
let hasInput = false;

const flags = mode === 'overwrite' ? 'w' : 'a';

const fileStream = fs.createWriteStream(filename, { flags });

// 4. Listen for incoming data
process.stdin.on('data', (chunk) => {
  hasInput = true;

  // Send input to the screen
  process.stdout.write(chunk);

  // Send input to the file
  fileStream.write(chunk);

  // Count complete lines
  const text = chunk.toString();
  lineCount += (text.match(/\n/g) || []).length;

  // Remember whether the last character was a newline
  lastChunkEndedWithNewline = text.endsWith('\n');
});

// 5. Listen for the end of input
process.stdin.on('end', () => {
  // Only add an extra line if there was input
  // and it did not end with a newline
  if (hasInput && !lastChunkEndedWithNewline) {
    lineCount++;
  }

  // Wait until the file is completely finished
  fileStream.end();
});

// Print summary only after the file is finished
fileStream.on('finish', () => {
  console.error(`Lines written: ${lineCount}`);
});

// 6. Handle errors
fileStream.on('error', (err) => {
  console.error(`Error writing to file: ${err.message}`);
  process.exit(1);
});

# Test file 
#!/bin/bash
# Test runner for fancyTee.js
# Usage: put this file next to fancyTee.js, then run:  bash test_fancyTee.sh

SCRIPT="$(pwd)/fancyTee.js"
WORK="$(mktemp -d)"
cd "$WORK" || exit 1
PASS=0
FAIL=0

# run "<input for printf>" <args for fancyTee.js...>
# Captures screen output, summary/errors (stderr) and exit code.
run() {
  local input="$1"; shift
  printf "$input" | node "$SCRIPT" "$@" >stdout.txt 2>stderr.txt
  CODE=$?
}

# check "<test name>" "<expected>" "<actual>"
check() {
  if [ "$2" == "$3" ]; then
    echo "  PASS: $1"; PASS=$((PASS+1))
  else
    echo "  FAIL: $1"
    echo "     expected: $2"
    echo "     actual:   $3"
    FAIL=$((FAIL+1))
  fi
}

lines_in() { wc -l < "$1" | tr -d ' '; }

echo "Test 1: normal input (3 lines)"
run "a\nb\nc\n" t1.txt
check "screen output matches input" "$(printf 'a\nb\nc\n')" "$(cat stdout.txt)"
check "file has 3 lines"            "3" "$(lines_in t1.txt)"
check "summary"                     "Lines written: 3" "$(cat stderr.txt)"
check "exit code"                   "0" "$CODE"

echo "Test 2: append (run the same command again)"
run "a\nb\nc\n" t1.txt
check "file now has 6 lines"        "6" "$(lines_in t1.txt)"
check "summary still counts 3"      "Lines written: 3" "$(cat stderr.txt)"

echo "Test 3: overwrite mode"
run "a\nb\nc\n" t1.txt overwrite
check "file has 3 lines"            "3" "$(lines_in t1.txt)"

echo "Test 4: no trailing newline"
run "a\nb" t4.txt
check "summary counts 2"            "Lines written: 2" "$(cat stderr.txt)"

echo "Test 5: empty input"
run "" t5.txt
check "summary counts 0"            "Lines written: 0" "$(cat stderr.txt)"

echo "Test 6: directory does not exist"
run "hi\n" nodir/out.txt
check "exit code 1"                 "1" "$CODE"
check "error message shown"         "yes" "$(grep -q 'Error writing' stderr.txt && echo yes || echo no)"
check "no summary printed"          "no" "$(grep -q 'Lines written' stderr.txt && echo yes || echo no)"

echo "Test 7: invalid mode"
run "hi\n" t7.txt badmode
check "exit code 1"                 "1" "$CODE"
check "mode error shown"            "yes" "$(grep -q 'Mode must be' stderr.txt && echo yes || echo no)"

echo "Test 8: no arguments"
run "hi\n"
check "exit code 1"                 "1" "$CODE"
check "usage shown"                 "yes" "$(grep -q 'Usage' stderr.txt && echo yes || echo no)"

echo "Test 9: filename with spaces"
run "x\ny\n" "my file.txt"
check "file created with 2 lines"   "2" "$(lines_in 'my file.txt')"

echo "Test 10: summary is not mixed into piped output"
run "a\nb\n" t10.txt
check "stdout has only the input"   "$(printf 'a\nb\n')" "$(cat stdout.txt)"

echo
echo "Results: $PASS passed, $FAIL failed"
rm -rf "$WORK"
