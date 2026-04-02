use DBIx::Wizard;

my @results = dbiw('library:books')
              ->as('b')
              ->join('authors|a' => 'a.name=b.author_name')
              ->find({ 'a.country' => 'FR' })
              ->all(['b.name|name')];

my @results = dbiw('ipdata:ip_ranges')
              ->as('i')
              ->join('autonomous_systems|a' => 'a.asn=i.asn')
              ->find({ dbiw->min('i.bla') => { '>' => 5 } })
              ->all(['ip')];

dbiw('library:books')->find({ author_name => 'Alan Turing' })
                     ->update({ views => dbiw->col('views') + 1 });

dbiw('library:books')->find({ author_name => 'Alan Turing' })
                     ->update({ last_update => dbiw->now });

