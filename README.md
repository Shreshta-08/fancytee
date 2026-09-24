# fancytee
fancytee.js
# Project Description
FancyTee is a Node.js command based on the Linux tee command. It takes input, displays it on the screen, saves it to a file, and counts the number of lines written.
# AI Reflection
I asked AI to help me understand tee, write my Node.js command, and find possible errors and edge cases.
AI helped me find a problem with no input, but I tested the code myself and fixed the issue.

#Code
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
