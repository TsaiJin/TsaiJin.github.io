---
title: The mechanism behind WriteBatch in leveldb
date: 2020-05-16 20:59:09
tags: [database, leveldb]
categories: Technology
---

### Introduction

As a well-known key-value database,[ leveldb](https://github.com/google/leveldb) provides general key-values interfaces like Put, Get and Delete. Besides those interfaces, leveldb provides a batch operation called WriteBatch as well. A batch operation means we can group multiple operations into one and submit this one to leveldb, the atomicity guarantees that either all of those operations are applied or none of them is applied.

For example, we can use batch operation in leveldb in the following way:

```C++
WriteBatch b;
batch.Put("key", "v1");
batch.Delete("key");
batch.Put("key", "v2");
batch.Put("key", "v3");
db->Write(WriteOptions(), &b);
```

The updates are applied in the order in which they are added to the WriteBatch. And the value of "key" in the above code sample will be "v3" after the batch is written.

### Implementaion of WriteBatch

Well, how does leveldb implement this simple but powerful interface? Let's figure it out by digging the source code step by step.

<!-- more -->



#### WriteBatch Class

The `WriteBatch` is a class containing member function `WriteBatch::Put`, `WriteBatch::Delete`,`WriteBatch::Clear` and `WriteBatch::Iterate`. A private member variable of string type called `rep_` is also owned by it.



In the constructor of `WriteBatch`, `rep_` is cleared firstly and resize to length `kHeader`:

```c++
static const size_t kHeader = 12;

WriteBatch::WriteBatch() {
  Clear();
}

void WriteBatch::Clear() {
  rep_.clear();
  rep_.resize(kHeader);
}
```

That's because the first `kHeader` bytes in `rep_` is used to maintain meta information: 8-byte sequence number and followed by a 4-byte count.



Each time when the user calls `WriteBatch::Put`, the `rep_` variable gets updated by the following behaviors :

```C++
void WriteBatch::Put(const Slice& key, const Slice& value) {
  WriteBatchInternal::SetCount(this, WriteBatchInternal::Count(this) + 1);
  rep_.push_back(static_cast<char>(kTypeValue));
  PutLengthPrefixedSlice(&rep_, key);
  PutLengthPrefixedSlice(&rep_, value);
}
```

Consequently, the 4-byte count in `rep_` gets incremented, and an enum char value `kTypeValue`(means this is a put operation), the key and the value are appended into the end of `rep_`.



Similarly, when the user calls `WriteBatch::Delete`, the `rep_` variable is updated in a similar way:

```C++
void WriteBatch::Delete(const Slice& key) {
  WriteBatchInternal::SetCount(this, WriteBatchInternal::Count(this) + 1);
  rep_.push_back(static_cast<char>(kTypeDeletion));
  PutLengthPrefixedSlice(&rep_, key);
}
```

Excepting that an enum char value `kTypeDeletion`(means this is a put operation) and the key information are appended into the `rep_`.



Let's be more specific about the function `PutLengthPrefixedSlice` called by both `WriteBatch::Delete`and `WriteBatch::Put`:

```C++
void PutLengthPrefixedSlice(std::string* dst, const Slice& value) {
  PutVarint32(dst, value.size());
  dst->append(value.data(), value.size());
}
```

It appends the value size into dst and then the value itself into dst.

Based on all the information provided above, we know that the `Put` and `Delete` member function of `WriteBatch` only update its member variable `rep_`. 

And we can also know the data layout format of `rep_`: 

```
WriteBatch::rep_ :=
      sequence: fixed64
      count: fixed32
      data: record[count]
record :=
      kTypeValue varstring varstring         |
      kTypeDeletion varstring
varstring :=
      len: varint32
      data: uint8[len]
```

{% asset_img rep_format.png image of rep_ format %}



#### DBImpl::Write

After a batch operation is encapsulated, it's time to call `DBImpl::Write` to apply the batch operation into the underlying layer to persist it.

Inside `DBImpl::Write`, an instance of `Writer` called `w` , which represents the batch operation context, is created and added to the end of the double-ended queue `writers_`.  Because write operation is in strict order, so the write thread must wait until `w` is the first element of `writers_`.

```c++
Writer w(&mutex_);
  w.batch = my_batch;
  w.sync = options.sync;
  w.done = false;

  MutexLock l(&mutex_);
  writers_.push_back(&w);
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait();
  }
  if (w.done) {
    return w.status;
  }
```



When the condition is fulfilled, leveldb will make sure there is enough room for this batch operation. Enough room mainly means memtable is prepared well to handle this write (I will write another new article to talk about that). Because `w` becomes the first element in `writers_`, the write thread will try to group other writers behind `w` into a so-called `BatchGroup` to speed up the io efficiency by the function `DBImpl::BuildBatchGroup`.

```c++
WriteBatch* DBImpl::BuildBatchGroup(Writer** last_writer) {
  assert(!writers_.empty());
  Writer* first = writers_.front();
  WriteBatch* result = first->batch;
  assert(result != NULL);

  size_t size = WriteBatchInternal::ByteSize(first->batch);

  // Allow the group to grow up to a maximum size, but if the
  // original write is small, limit the growth so we do not slow
  // down the small write too much.
  size_t max_size = 1 << 20; // 1M
  if (size <= (128<<10)) { // 128K
    max_size = size + (128<<10);
  }

  *last_writer = first;
  std::deque<Writer*>::iterator iter = writers_.begin();
  ++iter;  // Advance past "first"
  for (; iter != writers_.end(); ++iter) {
    Writer* w = *iter;
    if (w->sync && !first->sync) {
      // Do not include a sync write into a batch handled by a non-sync write.
      break;
    }

    if (w->batch != NULL) {
      size += WriteBatchInternal::ByteSize(w->batch);
      if (size > max_size) {
        // Do not make batch too big
        break;
      }

      // Append to *result
      if (result == first->batch) {
        // Switch to temporary batch instead of disturbing caller's batch
        result = tmp_batch_;
        assert(WriteBatchInternal::Count(result) == 0);
        WriteBatchInternal::Append(result, first->batch);
      }
      WriteBatchInternal::Append(result, w->batch);
    }
    *last_writer = w;
  }
  return result;
}
```

Inside `DBImpl::BuildBatchGroup`, it will traverse the whole `writers_` from beginning to end and try to group all  those batches into one batch. For convenience to clarify, we call this procedure "batch mergence"

However, there are some limitations in batch mergence. First, the maximum size of `rep_` in the merged batch group is **1M** and if the first writer is a small writer(`rep_.size() <= 128K` ) it will limit the maximum size to `rep_.size() + 128K`. The idea behind this is not slowing down the small write too much. Second, it will not include a sync write into the batch handled by a non-sync write.

In the batch mergence, with the help of `WriteBatchInternal::Append`, the data field of `rep_` in second batch is appended to the end of the `rep_` in first batch:

```c++
void WriteBatchInternal::Append(WriteBatch* dst, const WriteBatch* src) {
  SetCount(dst, Count(dst) + Count(src));
  assert(src->rep_.size() >= kHeader);
  dst->rep_.append(src->rep_.data() + kHeader, src->rep_.size() - kHeader);
}
```

After batch mergence is finished, a new batch called `updates` is generated. And the sequence number in `rep_` of `updates` is set to `last_sequence + 1`, `last_sequnce` gets adjusted to a new one(`last_sequnce = last_sequnce + count field rep_ of updates`) as well.



Now, we come to the IO critical path. The WAL in leveldb will persist the contents of the `update` in an append-only way. After that success, memtable will replay the operations in the batch `update` and do corresponding behavior in memtable:



Apply to WAL and insert into memtable:

```c++
{
      mutex_.Unlock();
      status = log_->AddRecord(WriteBatchInternal::Contents(updates));
      bool sync_error = false;
      if (status.ok() && options.sync) {
        status = logfile_->Sync();
        if (!status.ok()) {
          sync_error = true;
        }
      }
      if (status.ok()) {
        status = WriteBatchInternal::InsertInto(updates, mem_);
      }
      mutex_.Lock();
      if (sync_error) {
        // The state of the log file is indeterminate: the log record we
        // just added may or may not show up when the DB is re-opened.
        // So we force the DB into a mode where all future writes fail.
        RecordBackgroundError(status);
      }
    }
```



`WriteBatch` Parse the whole `rep_` string to replay original operations specified by users and call handler(MemTableInserter) to handle these operations.

```
Status WriteBatch::Iterate(Handler* handler) const {
  Slice input(rep_);
  if (input.size() < kHeader) {
    return Status::Corruption("malformed WriteBatch (too small)");
  }

  input.remove_prefix(kHeader);
  Slice key, value;
  int found = 0;
  while (!input.empty()) {
    found++;
    char tag = input[0];
    input.remove_prefix(1);
    switch (tag) {
      case kTypeValue:
        if (GetLengthPrefixedSlice(&input, &key) &&
            GetLengthPrefixedSlice(&input, &value)) {
          handler->Put(key, value);
        } else {
          return Status::Corruption("bad WriteBatch Put");
        }
        break;
      case kTypeDeletion:
        if (GetLengthPrefixedSlice(&input, &key)) {
          handler->Delete(key);
        } else {
          return Status::Corruption("bad WriteBatch Delete");
        }
        break;
      default:
        return Status::Corruption("unknown WriteBatch tag");
    }
  }
  if (found != WriteBatchInternal::Count(this)) {
    return Status::Corruption("WriteBatch has wrong count");
  } else {
    return Status::OK();
  }
}
```



### Summary

Hoo, finally get here! This article mainly talks about how batch operation is handled in leveldb. Due to space limitation, other import parts like WAL and Memtable in leveldb cannot be presented here.

Have fun~~~

