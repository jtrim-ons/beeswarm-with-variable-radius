<script>
	import chroma from 'chroma-js';
	import { getContext } from 'svelte';

	const { data, xGet, zGet, padding, width, height, config, custom } = getContext('LayerCake');

	export let r = 2;
	export let strokeWidth = 0;
	export let strokeColor = '#f00';
	export let spacing = .2;

    const colourScale = chroma
        .scale(['#A9BFC6', '#fa996a', '#cc0000'])
        .domain([-0.75, -0.5, -0.25]);

  $: series = [1,0].map(in_lockdown =>
        new AccurateBeeswarm($data.filter(d=>d.in_lockdown==in_lockdown), d => (d.r + .33) * (.5 + $width / 2000), $xGet)
			.withTiesBrokenRandomly()
//			.oneSided()
			.calculateYPositions()
			.map(d => ({x: d.x, [$config.z]: d.datum[$config.z], y: d.y+150+150*in_lockdown, data: d.datum})));

    class AccurateBeeswarm {
        constructor(items, radiusFun, xFun) {
            this.items = items;
            this.radiusFun = radiusFun;
            let radius = 2;
            this.diameter = radius * 2;
            this.diameterSq = this.diameter * this.diameter;
            this.xFun = xFun;
            this.tieBreakFn = x => x;
            this._oneSided = false;
            this.maxR = Math.max(...items.map(d => radiusFun(d)));
        }
    
        withTiesBrokenRandomly() {
            this.tieBreakFn = this._sfc32(0x9E3779B9, 0x243F6A88, 0xB7E15162, 1);
            return this;
        }
    
        oneSided() {
            this._oneSided = true;
            return this;
        }
    
        calculateYPositions() {
            let all = this.items.map((d,i) => ({
                datum: d,
                originalIndex: i,
                x: this.xFun(d),
                y: null,
                placed: false,
                forbiddenY: new UnionOfIntervals(),
                score: 0,
                bestPosition: 0,
                heapPos: -1
            })).sort((a,b) => a.x - b.x);
            all.forEach(function (d, i) {d.index = i;});
            let tieBreakFn = this.tieBreakFn;
            all.forEach(function(d) {d.tieBreaker = tieBreakFn(d.x)});
            let pq = new AccurateBeeswarmPriorityQueue(this.radiusFun);
            pq.push(...all);
            while (!pq.isEmpty()) {
                let item = pq.pop();
                item.placed = true;
                item.y = item.bestPosition;
                this._updateYBounds(item, all, pq);
            }
            all.sort((a, b) => a.originalIndex - b.originalIndex);
            return all.map(d => ({datum: d.datum, x: d.x, y: d.y}));
        }
    
        // Random number generator
        // https://stackoverflow.com/a/47593316
        _sfc32(a, b, c, d) {
            let rng = function() {
                a >>>= 0; b >>>= 0; c >>>= 0; d >>>= 0;
                var t = (a + b) | 0;
                a = b ^ b >>> 9;
                b = c + (c << 3) | 0;
                c = (c << 21 | c >>> 11);
                d = d + 1 | 0;
                t = t + d | 0;
                c = c + t | 0;
                return (t >>> 0) / 4294967296;
            }
            for (let i=0; i<10; i++) {
                rng();
            }
            return rng;
        }
    
        _updateYBounds(item, all, pq) {
            for (let step of [-1, 1]) {
                let xDist;
                let r = this.radiusFun(item.datum);
                for (let i = item.index + step;
                        i >= 0 &&
                            i < all.length &&
                            (xDist = Math.abs(item.x - all[i].x)) < r + this.maxR;
                        i += step) {
                    let other = all[i];
                    if (other.placed)
                        continue;
                    let sumOfRadii = r + this.radiusFun(other.datum);
                    if (xDist >= r + this.radiusFun(other.datum))
                        continue;
                    let yDist = Math.sqrt(sumOfRadii * sumOfRadii - xDist * xDist);
                    let forbiddenInterval = [item.y - yDist, item.y + yDist];
                    other.forbiddenY.addInterval(forbiddenInterval);
                    other.bestPosition = other.forbiddenY.bestAvailableValue(this._oneSided);
                    let prevScore = other.score;
                    other.score = Math.abs(other.bestPosition);
                    if (other.score > prevScore) {
                        // It is never the case that other.score < prevScore here.
                        // The if statement is unnecessary, but makes the algorithm
                        // run a little faster.
                        pq.deprioritise(other);
                    }
                }
            }
        }
    }
    
    class UnionOfIntervals {
        constructor() {
            this.head = {right: -Infinity};
            this.tail = {left: Infinity};
            this.head.next = this.tail;
            this.tail.prev = this.head;
        }
    
        del(i) {
            i.prev.next = i.next;
            i.next.prev = i.prev;
        }
    
        insertBefore(interval, i) {
            interval.prev = i.prev;
            interval.next = i;
            interval.prev.next = interval;
            i.prev = interval;
        }
    
        addInterval(interval) {
            let l = interval[0];
            let r = interval[1];
            interval = {left: l, right: r};
            let nxt;
            for (let i=this.head.next; i!=this.tail; i=nxt) {
                nxt = i.next;
                // if `i` contains `interval`, do nothing
                if (i.left <= l && i.right >= r) {
                    return;
                }
                // if `interval` contains `i`, delete `i`
                if (l <= i.left && r >= i.right) {
                    this.del(i);
                }
            }
            for (let i=this.head.next; i!=this.tail; i=nxt) {
                nxt = i.next;
                if (r <= i.left) {
                    this.insertBefore(interval, i);
                    return;
                }
                if (r > i.left && r < i.right) {
                    i.left = l;
                    return;
                }
                if (l < i.right) {
                    if (r > i.next.left) {
                        // combine i and i.next
                        i.right = i.next.right;
                        this.del(i.next);
                    } else {
                        i.right = r;
                    }
                    return;
                }
            }
            this.insertBefore(interval, this.tail);
        }
    
        bestAvailableValue(oneSided = false) {
            let best = Infinity;
            let zeroAvailable = true;
            for (let i=this.head.next; i!=this.tail; i=i.next) {
                if (i.left < 0 && i.right > 0)
                    zeroAvailable = false;
                if ((!oneSided || i.left >= 0) && Math.abs(i.left) < Math.abs(best))
                    best = i.left;
                if ((!oneSided || i.right >= 0) && Math.abs(i.right) < Math.abs(best))
                    best = i.right;
            }
            return zeroAvailable ? 0 : best;
        }
    
        toArray() {
            let result = [];
            for (let i=this.head.next; i!=this.tail; i=i.next) {
                result.push([i.left, i.right]);
            }
            return result;
        }
    }
    
    class AccurateBeeswarmPriorityQueue {
        // Based on https://stackoverflow.com/a/42919752
        parent(i) {return ((i + 1) >>> 1) - 1;}
        left(i) {return (i << 1) + 1;}
        right(i) {return (i + 1) << 1;}
    
        constructor(radiusFun) {
            this.TOP = 0;
            this._heap = [];
            this.radiusFun = radiusFun;
        }
        size() {
            return this._heap.length;
        }
        isEmpty() {
            return this.size() == 0;
        }
        peek() {
            return this._heap[this.TOP];
        }
        push(...values) {
            values.forEach(value => {
                value.heapPos = this.size();
                this._heap.push(value);
                this._siftUp();
            });
            return this.size();
        }
        pop() {
            const poppedValue = this.peek();
            const bottom = this.size() - 1;
            if (bottom > this.TOP) {
                this._swap(this.TOP, bottom);
            }
            this._heap.pop();
            this._siftDown();
            return poppedValue;
        }
        // Caution: this only works if new priority is less than or equal to old one.
        deprioritise(item) {
            this._siftDown(item.heapPos);
        }
        _greater(i, j) {
            let a = this._heap[i];
            let b = this._heap[j];
            let ra = this.radiusFun(a.datum) + a.tieBreaker * 5 - a.score * 0.1;
            let rb = this.radiusFun(b.datum) + b.tieBreaker * 5 - b.score * 0.1;
            if (ra > rb) return true;
            if (ra < rb) return false;
            if (a.score < b.score) return true;
            if (a.score > b.score) return false;
            return a.tieBreaker < b.tieBreaker;
        }
        _swap(i, j) {
            let tmp = this._heap[i];
            this._heap[i] = this._heap[j];
            this._heap[j] = tmp;
            this._heap[i].heapPos = i;
            this._heap[j].heapPos = j;
        }
        _siftUp() {
            let node = this.size() - 1;
            while (node > this.TOP && this._greater(node, this.parent(node))) {
                this._swap(node, this.parent(node));
                node = this.parent(node);
            }
        }
        _siftDown(node = this.TOP) {
            let l, r
            let sz = this.size();
            while (l = this.left(node), r = this.right(node),
                (l < sz && this._greater(l, node)) ||
                (r < sz && this._greater(r, node))
            ) {
                let maxChild = (r < sz && this._greater(r, l)) ? r : l;
                this._swap(node, maxChild);
                node = maxChild;
            }
        }
    }
</script>

<g class='bee-group'>
	{#each series as circles}
        {#each circles as d}
            <circle
                fill={colourScale(d.data.pct_change)}
                fill-opacity='0.8'
                stroke='{strokeColor}'
                stroke-width='{strokeWidth}'
                cx='{d.x}'
                cy='{$height - $padding.bottom - r - spacing - strokeWidth / 2 - d.y}'
                r='{d.data.r * (.5 + $width / 2000)}'
            >
                <title>{$custom.getTitle(d)}</title>
            </circle>
        {/each}
	{/each}
</g>
