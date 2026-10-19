# built-in-max-size

A one-file kit for Claude Code that checks a site measurement and works out the largest built-in that fits an opening: width, depth and height in mm.

The size depends not just on width, depth and height but on the wall angles and how far the walls are out of plumb. When built-ins are made to the maximum size with no extra allowance for scribing on site, a mistake in the measurement is expensive. Send the agent a measurement in any form (text, a table, a photo of a marked-up sketch). It fills a 51-field template, flags what doesn't add up before you start designing, and runs the numbers.

## Before you install

Read `built-in-max-size.md` yourself and have an AI check what it does. Do that with any file like this from the internet, this one included.

## Install

1. Download [`built-in-max-size.md`](built-in-max-size.md): open it and click the download button (Download raw file).
2. Put it into an empty folder.
3. Open Claude Code in that folder and type:

   ```
   Read built-in-max-size.md and set up the kit exactly as it says.
   ```

The agent creates the calculation script, the measurement template, an example, 19 tests and a field map, runs the tests and shows the example result: 1221 x 605 x 2607 mm. From then on, send it a measurement and check the result.

Needs Python 3.8 or newer. Nothing else. The kit also writes `AGENTS.md`, the same instructions for Codex and other agents that read that file; so far it has been tested in Claude Code only.

## How it was checked

The calculation is a plain Python script with no dependencies, ported one to one from the spreadsheet we use at work. It was checked against that spreadsheet on 41 measurements, an example plus 40 random ones, including blanks and edge cases: every intermediate value matched. How the numbers combine is in the kit's `README.md`.

The result is the envelope: fillers, scribe strips and clearances are not computed.

## License

MIT, see [LICENSE](LICENSE). Provided as is, without warranty: check every result before you cut.

## Author

Max Srebny, cabinetry CAD/CAM engineer. I build the software my industry doesn't have, with Claude Code. [LinkedIn](https://www.linkedin.com/in/max-srebny/)
