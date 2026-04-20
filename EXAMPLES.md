use DBIx::Wizard;

my @results = dbiw('library:books')
              ->as('b')
              ->join('authors|a' => 'a.name=b.author_name')
              ->find({ 'a.country' => 'FR' })
              ->all(['b.title|title')];

dbiw('library:books')->find({ author_name => 'Alan Turing' })
                     ->update({ views => dbiw->col('views') + 1 });

dbiw('library:books')->find({ author_name => 'Alan Turing' })
                     ->update({ last_update => dbiw->now });

