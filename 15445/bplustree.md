# b+ tree

持有header_page_代表这次操作可能改变root_page_id，`ctx->header_page_ = std::nullopt;`代表这次操作不会改变root_page_id。

由于没有对disk_manager持锁，一个page被evicted后在写入disk之前，可能再次被fetch后从disk中read，此时的数据是未被更新的，因此我们需要对dis_manager持锁，保证一个page在被读取前其disk中对应的数据已被更新。

我们可以对被删除的数据设置tombstone，当这个leafpage被split或者merge或者redistribute时，可以提前清除tombstone的数据，如果还是需要split或者merge，则按原流程进行。

