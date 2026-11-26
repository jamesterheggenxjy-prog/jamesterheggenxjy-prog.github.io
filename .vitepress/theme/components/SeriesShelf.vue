<template>
  <section v-if="books.length" class="series-shelf container" aria-label="系列目录">
    <div class="book-grid">
      <a
        v-for="book in books"
        :key="book.href"
        class="book-card"
        :href="withBase('/' + book.href + '#book-contents')"
      >
        <span class="book-spine" aria-hidden="true"></span>
        <div class="book-text">
          <h2>{{ book.bookTitle }}</h2>
          <p v-if="book.excerpt">{{ book.excerpt }}</p>
        </div>
        <span class="book-link">查看目录 <span aria-hidden="true">→</span></span>
      </a>
    </div>
  </section>
</template>

<script setup lang="ts">
import { withBase } from 'vitepress'
import { data as posts } from '../utils/posts.data'

const books = posts.filter((post) => post.bookTitle)
</script>

<style scoped lang="less">
.series-shelf {
  box-sizing: border-box;
  margin-bottom: 32px;
}

.book-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 360px), 1fr));
  gap: 16px;
}

.book-card {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr) auto;
  align-items: center;
  gap: 16px;
  padding: 19px 22px;
  border: 1px solid var(--pot-border-left);
  border-inline-start-width: 6px;
  border-radius: 18px;
  background: linear-gradient(115deg, rgba(var(--blue-shadow-color), 0.08), transparent 72%),
    var(--foreground-color);
  box-shadow: 0 2px 10px rgba(var(--blue-shadow-color), 0.18);
  color: var(--font-color-grey);
  transition: box-shadow var(--transition-time);

  &:hover,
  &:focus-visible {
    box-shadow: 0 3px 16px rgba(var(--blue-shadow-color), 0.36);
  }

  &:focus-visible {
    outline: 2px solid var(--icon-color);
    outline-offset: 3px;
  }
}

.book-spine {
  box-sizing: border-box;
  width: 36px;
  height: 44px;
  padding: 10px 7px;
  border-radius: 5px 8px 8px 5px;
  border-inline-start: 5px solid var(--pot-border-left);
  background-color: var(--btn-background);

  &::before {
    content: '';
    display: block;
    height: 6px;
    border-block: 1px solid var(--font-color-gold);
  }
}

.book-text {
  h2 {
    margin: 0;
    font-size: 20px;
  }

  p {
    margin: 5px 0 0;
    line-height: 1.45;
  }
}

.book-link {
  font-size: 14px;
  white-space: nowrap;

  span {
    margin-inline-start: 3px;
  }
}

@media (max-width: 768px) {
  .series-shelf {
    padding-inline: 16px;
  }
}

@media (max-width: 540px) {
  .book-card {
    grid-template-columns: auto minmax(0, 1fr);
    gap: 10px 14px;
    padding: 16px;
  }

  .book-link {
    grid-column: 2;
  }
}
</style>
