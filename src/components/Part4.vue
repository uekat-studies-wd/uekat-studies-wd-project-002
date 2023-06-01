<template>
    <div class="container">
        <h1 class="title">Możliwości tworzenia autorskich wykresów</h1>
        <div class="sigma-container"></div>
        <div id="buttons">
            <button id="random">
                <span>random</span>
            </button>
            <button id="circular">
                <span>circular</span>
            </button>
            <button id="spring-layout-on">
                <span>spring layout on</span>
            </button>
            <button id="spring-layout-off">
                <span>spring layout off</span>
            </button>
        </div>
        <div id="search">
            <input type="search" id="search-input" list="suggestions" placeholder="Try searching for a node..." />
            <datalist id="suggestions"></datalist>
        </div>
    </div>
</template>

<script lang="ts">
import Graph from "graphology";
import Sigma from "sigma";
import { circular } from "graphology-layout";
import { Coordinates, EdgeDisplayData, NodeDisplayData, PlainObject } from "sigma/types";
import { animateNodes } from "sigma/utils/animate";
import chroma from "chroma-js";
import { v4 as uuid } from "uuid";
import ForceSupervisor from "graphology-layout-force/worker";
import getNodeProgramImage from "sigma/rendering/webgl/programs/node.image";

export default {
    mounted() {
        // Initialize the graph object with data
        const graph = new Graph();
        graph.addNode("n1", { x: 0, y: 0, color: chroma.random().hex(), label: "New York", size: 25, type: "image", image: "/city.svg" });
        graph.addNode("n2", { x: -5, y: 3.3, color: chroma.random().hex(), label: "Los Angeles", size: 20, type: "image", image: "/user.svg" });
        graph.addNode("n3", { x: -5, y: 6.6, color: chroma.random().hex(), label: "Paris", size: 15 });
        graph.addNode("n4", { x: 0, y: 10, color: chroma.random().hex(), label: "Warsaw", size: 25 });
        graph.addNode("n5", { x: 5, y: 6.6, color: chroma.random().hex(), label: "Tokyo", size: 20 });
        graph.addNode("n6", { x: 5, y: 3.3, color: chroma.random().hex(), label: "Helsinki", size: 5 });
        graph.addEdge("n1", "n2", { size: 30, type: "line", label: "lorem" });
        graph.addEdge("n2", "n3", { size: 10, type: "arrow", label: "ipsum" });
        graph.addEdge("n3", "n4", { size: 5, type: "arrow" });
        graph.addEdge("n5", "n4", { size: 20, type: "arrow" });
        graph.addEdge("n5", "n6", { label: "lorem ipsum" });
        graph.addEdge("n6", "n1");
        // graph.import(data);

        graph.nodes().forEach((node, i) => {
            const angle = (i * 2 * Math.PI) / graph.order;
            graph.setNodeAttribute(node, "x", 100 * Math.cos(angle));
            graph.setNodeAttribute(node, "y", 100 * Math.sin(angle));
        });

        // Retrieve some useful DOM elements:
        const container = this.$el.querySelector(
            ".sigma-container",
        ) as HTMLElement;

        const randomButton = document.getElementById("random") as HTMLElement;
        const circularButton = document.getElementById("circular") as HTMLElement;
        const springLayoutOnButton = document.getElementById("spring-layout-on") as HTMLElement;
        const springLayoutOffButton = document.getElementById("spring-layout-off") as HTMLElement;

        // A variable is used to toggle state between start and stop
        let cancelCurrentAnimation: (() => void) | null = null;

        /** RANDOM LAYOUT **/
        /* Layout can be handled manually by setting nodes x and y attributes */
        /* This random layout has been coded to show how to manipulate positions directly in the graph instance */
        /* Alternatively a random layout algo exists in graphology: https://github.com/graphology/graphology-layout#random  */
        function randomLayout() {
            if (cancelCurrentAnimation) cancelCurrentAnimation();

            // to keep positions scale uniform between layouts, we first calculate positions extents
            const xExtents = { min: 0, max: 0 };
            const yExtents = { min: 0, max: 0 };
            graph.forEachNode((node, attributes) => {
                console.log(node);
                xExtents.min = Math.min(attributes.x, xExtents.min);
                xExtents.max = Math.max(attributes.x, xExtents.max);
                yExtents.min = Math.min(attributes.y, yExtents.min);
                yExtents.max = Math.max(attributes.y, yExtents.max);
            });
            const randomPositions: PlainObject<PlainObject<number>> = {};
            graph.forEachNode((node) => {
                // create random positions respecting position extents
                randomPositions[node] = {
                    x: Math.random() * (xExtents.max - xExtents.min),
                    y: Math.random() * (yExtents.max - yExtents.min),
                };
            });
            // use sigma animation to update new positions
            cancelCurrentAnimation = animateNodes(graph, randomPositions, { duration: 2000 });
        }

        // bind method to the random button
        randomButton.addEventListener("click", randomLayout);

        /** CIRCULAR LAYOUT **/
        /* This example shows how to use an existing deterministic graphology layout */
        function circularLayout() {
            if (cancelCurrentAnimation) cancelCurrentAnimation();

            //since we want to use animations we need to process positions before applying them through animateNodes
            const circularPositions = circular(graph, { scale: 100 });
            //In other context, it's possible to apply the position directly we : circular.assign(graph, {scale:100})
            cancelCurrentAnimation = animateNodes(graph, circularPositions, { duration: 2000, easing: "linear" });
        }

        // bind method to the random button
        circularButton.addEventListener("click", circularLayout);

        // Create the spring layout and start it
        const layout = new ForceSupervisor(graph, { isNodeFixed: (_, attr) => attr.highlighted });
        layout.start();

        // spring layout buttons
        springLayoutOnButton.addEventListener("click", () => { console.log('start'); layout.start() });
        springLayoutOffButton.addEventListener("click", () => { console.log('stop'); layout.stop() });

        /** instantiate sigma into the container **/
        // eslint-disable-next-line @typescript-eslint/no-unused-vars
        const renderer = new Sigma(graph, container, {
            nodeProgramClasses: {
                image: getNodeProgramImage(),
                // border: NodeProgramBorder,
            },
            renderEdgeLabels: true,
        });

        //
        // Drag'n'drop feature
        // ~~~~~~~~~~~~~~~~~~~
        //

        // State for drag'n'drop
        let draggedNode: string | null = null;
        let isDragging = false;

        // On mouse down on a node
        //  - we enable the drag mode
        //  - save in the dragged node in the state
        //  - highlight the node
        //  - disable the camera so its state is not updated
        renderer.on("downNode", (e) => {
            isDragging = true;
            draggedNode = e.node;
            graph.setNodeAttribute(draggedNode, "highlighted", true);
        });

        // On mouse move, if the drag mode is enabled, we change the position of the draggedNode
        renderer.getMouseCaptor().on("mousemovebody", (e) => {
            if (!isDragging || !draggedNode) return;

            // Get new position of node
            const pos = renderer.viewportToGraph(e);

            graph.setNodeAttribute(draggedNode, "x", pos.x);
            graph.setNodeAttribute(draggedNode, "y", pos.y);

            // Prevent sigma to move camera:
            e.preventSigmaDefault();
            e.original.preventDefault();
            e.original.stopPropagation();
        });

        // On mouse up, we reset the autoscale and the dragging mode
        renderer.getMouseCaptor().on("mouseup", () => {
            if (draggedNode) {
                graph.removeNodeAttribute(draggedNode, "highlighted");
            }
            isDragging = false;
            draggedNode = null;
        });

        // Disable the autoscale at the first down interaction
        renderer.getMouseCaptor().on("mousedown", () => {
            if (!renderer.getCustomBBox()) renderer.setCustomBBox(renderer.getBBox());
        });

        //
        // Create node (and edge) by click
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
        //

        // When clicking on the stage, we add a new node and connect it to the closest node
        renderer.on("clickStage", ({ event }: { event: { x: number; y: number } }) => {
            // Sigma (ie. graph) and screen (viewport) coordinates are not the same.
            // So we need to translate the screen x & y coordinates to the graph one by calling the sigma helper `viewportToGraph`
            const coordForGraph = renderer.viewportToGraph({ x: event.x, y: event.y });

            // We create a new node
            const node = {
                ...coordForGraph,
                size: 10,
                color: chroma.random().hex(),
            };

            // Searching the two closest nodes to auto-create an edge to it
            const closestNodes = graph
                .nodes()
                .map((nodeId) => {
                    const attrs = graph.getNodeAttributes(nodeId);
                    const distance = Math.pow(node.x - attrs.x, 2) + Math.pow(node.y - attrs.y, 2);
                    return { nodeId, distance };
                })
                .sort((a, b) => a.distance - b.distance)
                .slice(0, 2);

            // We register the new node into graphology instance
            const id = uuid();
            graph.addNode(id, node);

            // We create the edges
            closestNodes.forEach((e) => graph.addEdge(id, e.nodeId));
        });

        const searchInput = document.getElementById("search-input") as HTMLInputElement;
        const searchSuggestions = document.getElementById(
            "suggestions"
        ) as HTMLDataListElement;

        // Type and declare internal state:
        interface State {
            hoveredNode?: string;
            searchQuery: string;

            // State derived from query:
            selectedNode?: string;
            suggestions?: Set<string>;

            // State derived from hovered node:
            hoveredNeighbors?: Set<string>;
        }
        const state: State = { searchQuery: "" };

        // Feed the datalist autocomplete values:
        searchSuggestions.innerHTML = graph
            .nodes()
            .map(
                (node) =>
                    `<option value="${graph.getNodeAttribute(node, "label")}"></option>`
            )
            .join("\n");

        // Actions:
        function setSearchQuery(query: string) {
            state.searchQuery = query;

            if (searchInput.value !== query) searchInput.value = query;

            if (query) {
                const lcQuery = query.toLowerCase();
                const suggestions = graph
                    .nodes()
                    .map((n) => ({
                        id: n,
                        label: graph.getNodeAttribute(n, "label") as string
                    }))
                    .filter(({ label }) => label?.toLowerCase().includes(lcQuery));

                // If we have a single perfect match, them we remove the suggestions, and
                // we consider the user has selected a node through the datalist
                // autocomplete:
                if (suggestions.length === 1 && suggestions[0].label === query) {
                    state.selectedNode = suggestions[0].id;
                    state.suggestions = undefined;

                    // Move the camera to center it on the selected node:
                    const nodePosition = renderer.getNodeDisplayData(
                        state.selectedNode
                    ) as Coordinates;
                    renderer.getCamera().animate(nodePosition, {
                        duration: 500
                    });
                }
                // Else, we display the suggestions list:
                else {
                    state.selectedNode = undefined;
                    state.suggestions = new Set(suggestions.map(({ id }) => id));
                }
            }
            // If the query is empty, then we reset the selectedNode / suggestions state:
            else {
                state.selectedNode = undefined;
                state.suggestions = undefined;
            }

            // Refresh rendering:
            renderer.refresh();
        }
        function setHoveredNode(node?: string) {
            if (node) {
                state.hoveredNode = node;
                state.hoveredNeighbors = new Set(graph.neighbors(node));
            } else {
                state.hoveredNode = undefined;
                state.hoveredNeighbors = undefined;
            }

            // Refresh rendering:
            renderer.refresh();
        }

        // Bind search input interactions:
        searchInput.addEventListener("input", () => {
            setSearchQuery(searchInput.value || "");
        });
        searchInput.addEventListener("blur", () => {
            setSearchQuery("");
        });

        // Bind graph interactions:
        renderer.on("enterNode", ({ node }) => {
            setHoveredNode(node);
        });
        renderer.on("leaveNode", () => {
            setHoveredNode(undefined);
        });

        // Render nodes accordingly to the internal state:
        // 1. If a node is selected, it is highlighted
        // 2. If there is query, all non-matching nodes are greyed
        // 3. If there is a hovered node, all non-neighbor nodes are greyed
        renderer.setSetting("nodeReducer", (node, data) => {
            const res: Partial<NodeDisplayData> = { ...data };

            if (
                state.hoveredNeighbors &&
                !state.hoveredNeighbors.has(node) &&
                state.hoveredNode !== node
            ) {
                res.label = "";
                res.color = "#f6f6f6";
            }

            if (state.selectedNode === node) {
                res.highlighted = true;
            } else if (state.suggestions && !state.suggestions.has(node)) {
                res.label = "";
                res.color = "#f6f6f6";
            }

            return res;
        });

        // Render edges accordingly to the internal state:
        // 1. If a node is hovered, the edge is hidden if it is not connected to the
        //    node
        // 2. If there is a query, the edge is only visible if it connects two
        //    suggestions
        renderer.setSetting("edgeReducer", (edge, data) => {
            const res: Partial<EdgeDisplayData> = { ...data };

            if (state.hoveredNode && !graph.hasExtremity(edge, state.hoveredNode)) {
                res.hidden = true;
            }

            if (
                state.suggestions &&
                (!state.suggestions.has(graph.source(edge)) ||
                    !state.suggestions.has(graph.target(edge)))
            ) {
                res.hidden = true;
            }

            return res;
        });

    },
};
</script>

<style scoped>
.sigma-container {
    margin-bottom: 1rem;
}

#buttons {
    position: relative;
    display: flex;
}

#buttons>button {
    margin-right: 1em;
    display: inline-block;
    text-align: center;
    background: white;
    outline: none;
    border: 1px solid dimgrey;
    border-radius: 2px;
    cursor: pointer;
}

#buttons>button img {
    height: 2em;
}

#buttons>button>span {
    height: 100%;
    display: flex;
    align-items: center;
}

#buttons>button:last-child {
    margin-right: 0;
}

#search {
    position: relative;
    margin-top: 1rem;
}
</style>
