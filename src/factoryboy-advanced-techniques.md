---
title: "Factory Boy: Advanced Techniques"
revealjs:
    height: 850
    width: 1050
---

# Factory Boy

---

#### What does it do?

- It's a data fixture generation library for Python
- We use it to create semi-random test data

---

#### Simple usage

<pre><code data-trim class="language-python" data-line-numbers="1-2|4-8|10-11|13-14">
class Book(models.Model):
    name = models.CharField()

class BookFactory(factory.DjangoModelFactory):
    name = factory.FuzzyText()

    class Meta:
        model = "libraries.Book"

# create a singular record
book = BookFactory.create()

# create multiple
books = BookFactory.create_batch(100)
</code></pre>

---

#### Complex data modeling

<pre><code data-trim class="language-python" data-line-numbers="1-2|4-6|8-14|11-14">
class Author(models.Model):
    name = models.CharField()

class Topic(models.Model):
    slug = models.CharField(unqiue=True)
    name = models.CharField()

class Book(models.Model):
    title = models.CharField()
    author = models.ForeignKey(Author)
    topics = models.ManyToMany(
        Topic,
        related_name="books"
    )
    sequel = models.ForeignKey(
        "Book",
        null=True,
        on_delete=models.SET_NULL
    )
</code></pre>

---

#### Faker for randomness

<!-- .slide: data-auto-animate -->

<pre data-id="code-animation-topicfactory"><code data-trim class="language-python" data-line-numbers="">
class TopicFactory(factory.DjangoModelFactory):
    code = factory.Faker("word")
    name = factory.Faker("word")
</code></pre>

---

#### Make it less random

<!-- .slide: data-auto-animate -->

<pre data-id="code-animation-topicfactory"><code data-trim class="language-python" data-line-numbers="3-5">
class TopicFactory(factory.DjangoModelFactory):
    slug = factory.Faker("word")
    name = factory.LazyAttribute(
        lambda topic: topic.code.capitalize()
    )
</code></pre>

Note:
- More realistic relationship between the two values

---

#### Random with real-world values

<!-- .slide: data-auto-animate -->

<pre data-id="code-animation-topicfactory"><code data-trim class="language-python" data-line-numbers="1-5|8|9-11|15">
TOPICS = {
    topic["slug"]: topic["name"] for topic in json.loads(
        Path("test_data/topics.json").read_text()
    )
}

class TopicFactory(factory.DjangoModelFactory):
    slug = factory.fuzzy.FuzzyChoice(TOPICS.keys())
    name = factory.LazyAttribute(
        lambda topic: TOPICS[topic.slug]
    )

    class Meta:
        model = Topic
        django_get_or_create = ("slug",)
</code></pre>

Note:
- Limits the quantity of records so that topics will get reused across multiple books
- The data is more realistic, which makes debugging failed test output easier
- Informs factoryboy to use `get_or_create`

---

#### Many-to-many relationships

<!-- .slide: data-auto-animate -->

<pre data-id="code-animation-bookfactory"><code data-trim class="language-python" data-line-numbers="3-8|10">
class BookFactory(factory.DjangoModelFactory):

    @factory.postgeneration
    def topics(self, create, extracted, **kwargs):
        if not create or not extracted:
            return

        self.topics.set(extracted)

BookFactory.create(topics=TopicFactory.create_batch(3))
</code></pre>

---

#### Generate random related data

<!-- .slide: data-auto-animate -->

<pre data-id="code-animation-bookfactory"><code data-trim class="language-python" data-line-numbers="8-11|15|16|17">
class BookFactory(factory.DjangoModelFactory):

   @factory.post_generation
    def topics(self, create, extracted, **kwargs):
        if not create or not extracted:
            return

        if not extracted:
            extracted = TopicFactory.create_batch(
                random.randint(0,3)
            )

        self.topics.set(extracted)

BookFactory.create() # gets 1-3 topics
BookFactory.create(topics=[TopicFactory.create()]) # 1 topic
BookFactory.create(topics=[]) # oops, 1-3 topics
</code></pre>

---

#### Traits

<!-- .slide: data-auto-animate -->

<pre data-id="code-animation-bookfactory"><code data-trim class="language-python" data-line-numbers="6-7|18-19|21-22|24-25|27">
class BookFactory(factory.DjangoModelFactory):

   @factory.post_generation
    def topics(self, create, extracted, **kwargs):
        if not create or (
            isinstance(extracted, list | tuple)
            and len(extracted) == 0
        ):
            return

        if not extracted:
            extracted = TopicFactory.create_batch(
                random.randint(0,3)
            )

        self.topics.set(extracted)

    class Params:
        no_topics = factory.Trait(topics=[])

BookFactory.create(topics=[])
BookFactory.create(no_topics=True)

class NoTopicsBookFactory(BookFactory):
    no_topics = True

NoTopicsBookFactory.create()
</code></pre>
---

#### Maybes

<!-- .slide: data-auto-animate -->

<pre data-id="code-animation-bookfactory"><code data-trim class="language-python" data-line-numbers="2-9|12-16|18|19|20">
class BookFactory(factory.DjangoModelFactory):
    sequel = factory.Maybe(
        factory.Faker("pybool"),
        yes_declaration=SubFactory(
            BookFactory,
            sequel=factory.SubFactory(BookFactory)
        ),
        no_declaration=None
    )

    class Params:
        has_sequel = factory.Trait(
            sequel=factory.SubFactory(
                "libraries.factories.BookFactory",
            )
        )

BookFactory.create()
BookFactory.create(has_sequel=True, sequel__has_sequel=True)
BookFactory.create(has_sequel=False)
</code></pre>
