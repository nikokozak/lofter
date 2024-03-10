<script>

    import * as d3 from 'd3';
    //TODO: 
    // bring in D3;

    const inchParser = (inch_string) => {
        // Takes a string representing an inch measurement.
        // Returns inch-decimal or -1;
        // TODO: edge cases and errors.

        const reg = /(?<base>[0-9]+)(?<fraction>\s+(?<numerator>[0-9]+)\/(?<divisor>[0-9]+))?/g;
        const run = reg.exec(inch_string);

        if (run) {
            const { base, fraction, numerator, divisor } = run.groups;
            return +base + ((numerator ? +numerator : 0) / (divisor ? +divisor : 1));
        } else {
            return -1; 
        }
    }

    console.log(`inchParser: ${inchParser("21 3/8\"")}`);

    const loftParser = (loft_measure_string) => {
        // Takes a X-X-X(+/-) string value;
        // Returns inch-decimal or -1;

        const reg = /(?<feet>[0-9]+)-(?<inches>[0-9]+)-(?<eigths>[0-9])(?<sixteenths>[+-])?/g;
        const matched = reg.exec(loft_measure_string);

        if (matched) {
            const { feet, inches, eigths } = matched.groups;
            let { sixteenths } = matched.groups;

            sixteenths = sixteenths ? (sixteenths == "+" ? 1/16 : -1/16) : 0;
            
            return (+feet * 12) + +inches + +eigths/8 + sixteenths;
        } else {
            return -1; 
        }

    }

    console.log(`loftParser: ${loftParser("1-5-2")}`);

    const data = {
        WaterlineLabels: ["Keel", "Rabbet", 3, 6, 12, 15, 18, 21, 24, "Sheer"],
        ButtockLabels: ["Keel", "Rabbet", 6, 12, 18, 24, "Sheer"],
        DiagLines: ["A", "B"], // AFT RABBET AND FWD RABBET ADDED TO STATIONS LIST
        Stations: [0.5, 1, 2, 3, 4, 5, 6, 7, 7.5],
        StationSpacing: "21 3/8\"",
        ButtockHeights: [
            ["0-1-2", "0-2-2+", "0-0-1+", "0-0-0+", "0-0-0", "0-0-0+", "0-0-1", "0-0-2", "0-1-4+"], // Keel
            ["0-3-3+", "0-1-7+", "0-1-4", "0-1-2+", "0-1-2+", "0-1-3", "0-1-3+", "0-1-5+", "0-3-4+"], // Rabbet
            ["1-5-2", "0-7-3", "0-3-0", "0-2-0+", "0-1-7", "0-2-1", "0-3-0+", "0-7-4", "1-7-4"], // 6
            ["-", "1-6-7", "0-5-6", "0-3-4", "0-3-0", "0-3-3+", "0-6-0", "1-7-7", "-"], // 12
            ["-", "-", "0-11-3+", "0-6-2", "0-5-0", "0-5-6+", "1-0-3", "-", "-"], // 18
            ["-", "-", "-", "-", "1-1-3+", "-", "-", "-", "-"], // 24
            ["2-0-3", "1-10-6", "1-8-0+", "1-6-6", "1-6-1", "1-6-4", "1-7-5+", "1-10-2+", "1-11-7+"], // Sheer
        ],
        WaterlineHalfbreadths: [
            ["0-0-2+", "0-1-0", "0-1-2", "0-1-2", "0-1-2", "0-1-2", "0-1-2", "0-1-0", "0-0-2+"], // Keel
            ["0-1-0", "0-1-0", "0-1-2", "0-1-2", "0-1-2", "0-1-2", "0-1-2", "0-1-0", "0-1-0"], // Rabbet
            ["-", "0-2-0+", "0-6-0", "0-10-2", "0-11-7", "0-10-3+", "0-5-6+", "0-2-2", "-"], // 3
            ["0-2-0", "0-4-7", "1-0-3+", "1-5-5+", "1-7-3+", "1-6-2", "1-0-0", "0-4-5", "0-1-6+"], // 6
            ["0-3-0+", "0-7-2+", "1-4-2", "1-8-5+", "1-10-2", "1-8-7", "1-3-5+", "0-7-0", "0-2-7"], // 9 
            ["0-4-2", "0-9-4", "1-6-2", "1-10-3", "1-11-4+", "1-10-2+", "1-5-6", "0-9-0", "0-3-7+"], // 12
            ["0-5-3", "0-10-7+", "1-7-2+", "1-11-2+", "2-0-3+", "1-11-1+", "1-7-0", "0-10-4", "0-4-7+"], // 15
            ["0-6-1", "0-11-6+", "1-7-7+", "1-11-7", "2-1-0", "1-11-6+", "1-7-6+", "0-11-4+", "0-5-5+"], // 18
            ["0-6-4", "1-0-2+", "-", "-", "-", "-", "-", "1-0-2", "0-6-1+"], // 21
            ["0-6-5+", "-", "-", "-", "-", "-", "-", "-", "-"], // 24
            ["0-6-5+", "1-0-3", "1-8-2", "1-11-7+", "2-1-0", "1-11-7+", "1-8-0+", "1-0-3", "0-6-4"], // Sheer
        ],
        Diagonals: [
            ["0-1-2", "0-7-7", "1-2-0", "1-9-7+", "2-1-4+", "2-2-6+", "2-1-6", "1-9-4", "1-1-4+", "0-7-3+", "0-1-2"], // A
            ["0-1-5+", "0-9-5", "1-3-3+", "1-9-7+", "2-0-3+", "2-1-1", "2-0-5+", "1-9-5+", "1-3-0", "0-9-0", "0-1-5+"] // B
        ]
    }

    const buttockLines = (buttockHeights, buttockLabels, stations) => {
        return buttockHeights.map((lineHeights, lineIdx) => {
            return {
                buttock: buttockLabels[lineIdx],
                heights: lineHeights.map((height, heightIdx) => {
                    const decimalHeight = loftParser(height);
                    if (decimalHeight != -1) {
                        return {
                            height: decimalHeight,
                            station: stations[heightIdx]
                        }
                    }
                }).filter(x => x != undefined)
            }
        })
    }


    const width = 800;
    const height = 400;
    const marginX = 20;
    const marginY = 20;

    const buttockHeights = buttockLines(data.ButtockHeights, data.ButtockLabels, data.Stations);
    console.log(buttockHeights);


    // StationSpacing is irrelevant here because we have to keep a fixed aspect ratio for the whole thing to make sense.
    const xProfileScale = d3.scaleLinear([0, data.Stations[data.Stations.length - 1]], [marginX, width - marginX]);
    const yProfileScale = d3.scaleLinear([0, 26], [height - marginY, marginY]);

    const line = d3.line().x(x => xProfileScale(x.station))
    .y(x => yProfileScale(x.height)).curve(d3.curveCatmullRom.alpha(0.5));

    console.log(buttockHeights[2].heights.map(x => x.height))
    console.log(buttockHeights[2].heights.map(x => x.station))
    console.log(`line: ${line(buttockHeights[2].heights)}`)
</script>

<svg {width} {height}>
    <!-- X Axis -->
    <line x1={marginX} y1={height-marginY} x2={width-marginX} y2={height-marginY} stroke="black"/>

    <!-- Y Axis -->
    <line x1={marginX} y1={height-marginY} x2={marginX} y2={marginY} stroke="black" fill="none"/>

    {#each buttockLines(data.ButtockHeights, data.ButtockLabels, data.Stations) as bline} 
        <g desc={bline.buttock}>
            <path stroke="red" d={line(bline.heights)} fill="none"/>
            {#each bline.heights as point}
                <circle cx={xProfileScale(point.station)} cy={yProfileScale(point.height)} r="2" />
            {/each}
        </g>
    {/each}
</svg>

<h1>Heights above Baseline</h1>
<table>
    <caption>Halfbreadths</caption>
    <thead>
        <tr>
            <th></th> <!-- Empty cell -->
            {#each data.Stations as station}
                <th scope="col">
                    STA {station}
                </th>
            {/each}
        </tr>
    </thead> 
    <tbody>
        {#each data.ButtockLabels as buttock, i}
            <tr>
                <th scope="row">
                    {#if typeof buttock == 'string'}
                        {buttock}
                    {:else}
                        Buttock {buttock}
                    {/if}
                </th>
                {#each data.ButtockHeights[i] as halfbreadth}
                    <td>{halfbreadth}</td>
                {/each}
            </tr>
        {/each}
    </tbody>
</table>

<table>
    <caption>Halfbreadths from CenterLine</caption>
    <thead>
        <tr>
            <th></th> <!-- Empty cell -->
            {#each data.Stations as station}
                <th scope="col">
                    STA {station}
                </th>
            {/each}
        </tr>
    </thead> 
    <tbody>
        {#each data.WaterlineLabels as waterline, i}
            <tr>
                <th scope="row">
                    {#if typeof waterline == 'string'}
                        {waterline}
                    {:else}
                        Waterline {waterline}
                    {/if}
                </th>
                {#each data.WaterlineHalfbreadths[i] as height}
                    <td>{height}</td>
                {/each}
            </tr>
        {/each}
    </tbody>
</table>

<table>
    <caption>Diagonals</caption>
    <thead>
        <tr>
            <th></th> <!-- Empty cell -->
            {#each ["FWD RABBET", ...data.Stations, "AFT RABBET"] as waterline}
                <th scope="col">
                    {#if typeof waterline == 'string'}
                        {waterline}
                    {:else}
                        STA {waterline}
                    {/if}
                </th>
            {/each}
        </tr>
    </thead> 
    <tbody>
        {#each data.DiagLines as diag, i}
            <tr>
                <th scope="row">{diag}</th>
                {#each data.Diagonals[i] as diagonal}
                    <td>{diagonal}</td>
                {/each}
            </tr>
        {/each}
    </tbody>
</table>

