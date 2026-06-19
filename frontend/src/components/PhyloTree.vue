<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watch, nextTick } from 'vue';
import type { PhyloNode } from '../types';

const props = defineProps<{
  tree: PhyloNode | null;
}>();

const canvasRef = ref<HTMLCanvasElement | null>(null);
const containerRef = ref<HTMLDivElement | null>(null);

let resizeObserver: ResizeObserver | null = null;

function countLeaves(node: PhyloNode): number {
  if (node.children.length === 0) return 1;
  return node.children.reduce((sum, child) => sum + countLeaves(child), 0);
}

function getMaxDepth(node: PhyloNode, current = 0): number {
  if (node.children.length === 0) return current + node.branchLength;
  return Math.max(...node.children.map(c => getMaxDepth(c, current + node.branchLength)));
}

function drawTree() {
  const canvas = canvasRef.value;
  const container = containerRef.value;
  if (!canvas || !props.tree) return;

  const containerWidth = container ? container.clientWidth - 32 : 500;
  const totalLeaves = countLeaves(props.tree);
  const minHeight = Math.max(300, totalLeaves * 30 + 60);
  const containerHeight = container ? Math.max(container.clientHeight - 80, minHeight) : minHeight;
  const width = Math.max(containerWidth, 400);
  const height = Math.max(containerHeight, 300);
  canvas.width = width;
  canvas.height = height;

  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  // Background
  ctx.fillStyle = '#111827';
  ctx.fillRect(0, 0, width, height);

  const maxDepth = getMaxDepth(props.tree) || 1;

  const leftMargin = 30;
  const rightMargin = 140;
  const topMargin = 30;
  const bottomMargin = 30;
  const treeWidth = width - leftMargin - rightMargin;
  const treeHeight = height - topMargin - bottomMargin;

  let leafIndex = 0;
  const xScale = treeWidth / maxDepth;
  const yStep = treeHeight / Math.max(totalLeaves - 1, 1);

  // Recursive draw function
  function drawNode(node: PhyloNode, x: number, y: number): { x: number; y: number } {
    const nodeX = x + node.branchLength * xScale;

    if (node.children.length === 0) {
      // Leaf node
      const leafY = topMargin + leafIndex * yStep;
      leafIndex++;

      // Draw branch
      ctx.strokeStyle = '#6ee7b7';
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(x, leafY);
      ctx.lineTo(nodeX, leafY);
      ctx.stroke();

      // Draw leaf label
      ctx.fillStyle = '#d1d5db';
      ctx.font = '11px sans-serif';
      ctx.textAlign = 'left';
      const label = node.name.length > 20 ? node.name.substring(0, 18) + '...' : node.name;
      ctx.fillText(label, nodeX + 8, leafY + 4);

      // Draw dot
      ctx.fillStyle = '#06b6d4';
      ctx.beginPath();
      ctx.arc(nodeX, leafY, 3, 0, Math.PI * 2);
      ctx.fill();

      return { x: nodeX, y: leafY };
    }

    // Internal node - draw children first
    const childPositions = node.children.map(child => drawNode(child, nodeX, y));

    // Calculate this node's Y as the midpoint of children
    const minY = Math.min(...childPositions.map(p => p.y));
    const maxY = Math.max(...childPositions.map(p => p.y));
    const nodeY = (minY + maxY) / 2;

    // Draw vertical line connecting children
    ctx.strokeStyle = '#6ee7b7';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(nodeX, minY);
    ctx.lineTo(nodeX, maxY);
    ctx.stroke();

    // Draw horizontal branch to parent
    if (x > 0 || node.branchLength > 0) {
      ctx.beginPath();
      ctx.moveTo(x, nodeY);
      ctx.lineTo(nodeX, nodeY);
      ctx.stroke();
    }

    // Draw internal node dot
    ctx.fillStyle = '#a78bfa';
    ctx.beginPath();
    ctx.arc(nodeX, nodeY, 2.5, 0, Math.PI * 2);
    ctx.fill();

    return { x: nodeX, y: nodeY };
  }

  // Start drawing from root
  drawNode(props.tree, leftMargin, topMargin);

  // Title
  ctx.fillStyle = '#9ca3af';
  ctx.font = 'bold 12px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText('进化树 (Neighbor-Joining)', width / 2, 18);
}

onMounted(() => {
  drawTree();
  if (containerRef.value) {
    resizeObserver = new ResizeObserver(() => {
      nextTick(() => drawTree());
    });
    resizeObserver.observe(containerRef.value);
  }
});

onBeforeUnmount(() => {
  if (resizeObserver) {
    resizeObserver.disconnect();
    resizeObserver = null;
  }
});

watch(() => props.tree, () => {
  nextTick(() => drawTree());
}, { deep: true });
</script>

<template>
  <div ref="containerRef" class="bg-gray-900 rounded-lg overflow-hidden w-full">
    <div class="px-4 py-2 bg-gray-800 border-b border-gray-700 flex items-center justify-between">
      <h3 class="text-sm font-semibold text-gray-300">系统发育树</h3>
      <span class="text-xs text-gray-500">Neighbor-Joining 法</span>
    </div>
    <div class="p-4 w-full overflow-auto">
      <canvas
        v-if="tree"
        ref="canvasRef"
        class="border border-gray-700 rounded w-full"
      ></canvas>
      <div v-else class="flex items-center justify-center w-full h-64 text-gray-600 text-sm">
        请先加载序列并构建进化树
      </div>
    </div>
  </div>
</template>
