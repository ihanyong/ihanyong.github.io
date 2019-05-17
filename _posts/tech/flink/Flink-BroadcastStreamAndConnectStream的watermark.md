Flink-BroadcastStreamAndConnectStream的watermark.md



对于多流合并，
- 在合并节点上的watermark是所有输入流的最小的watermark。 
- 在eventtime中，若是有一个流没有赋watermark， 则合并节点上的watermark会一直滞留在Long.MIN_VALUE
- 也可以在合并节点后emit watermark  进行处理

因为broadcastState 也是通过流的形式来connect到主线流上的， 也是一个输入注，在使用 eventtime 时需要注意 watermark的问题。
