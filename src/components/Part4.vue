<template>
    <div class="container">
        <h1 class="title">Możliwości tworzenia autorskich wykresów</h1>
        <div class="sigma-container"></div>
    </div>
</template>

<script lang="ts">
import Graph from "graphology";
import Sigma from "sigma";

export default {
    mounted() {
        const container = this.$el.querySelector(
            ".sigma-container"
        ) as HTMLElement;

        const graph = new Graph();

        graph.addNode("John", {
            x: 0,
            y: 10,
            size: 5,
            label: "John",
            color: "blue",
        });
        graph.addNode("Mary", {
            x: 10,
            y: 0,
            size: 3,
            label: "Mary",
            color: "red",
        });

        graph.addEdge("John", "Mary");

        new Sigma(graph, container);
    },
};
</script>

<style scoped></style>
